# ruby的sudo gem

FileUtils操作普通文件没有问题，但如果需要root权限的，就不行.

gem 'sudo'解决了这个问题.

可以以  sudo[FileUtils].mv(source,target) 调用方式.

但有个问题就是: 进程和效率的问题.

按照文档，必须先调用 start!, 然后 stop!来关闭.
这样一来，每次调用 sudo[FileUtils]都要开启一个进程，效率太差.

而我们在实现一些方法时，是会不断调用的.

```ruby
  def link_dir(source:, target:, use_sudo: false)
    unless directory?(source, use_sudo: use_sudo)
      raise "source #{source} is not directory"
    end
    pre_outfile(out_file: target, keep_origin: false, use_sudo: use_sudo)
    FileUtils.ln_s(source, target, force: true)
  rescue Errno::EACCES
    raise unless use_sudo
    sudo = get_sudo_wrapper
    sudo[FileUtils].ln_sf(source, target)
  end
```
类似这个创建link_dir的方法。里面就需要调用sudo有三处之多.
就算每个方法都用 Sudo::Wrapper.run 也有问题.
因为，每个方法都调用，也是多个进程在跑.

这时，大家想到的就是，能否共享最外围的sudo，然后就避免的多次重复触发进程的创建。
这是可以的.

```ruby
class SudoHelper
  def self.run(&block)
    return if block.nil?
    sudo_wrapper = Thread.current.thread_variable_get(:__sudo_wrapper)
    if sudo_wrapper.nil?
      Sudo::Wrapper.run |sudo_wrapper|
         Thread.current.thread_variable_set(:__sudo_wrapper, sudo_wrapper)
         block.call(sudo_wrapper)
      ensure
         Thread.current.thread_variable_set(:__sudo_wrapper, nil)
      end
    else
      block.call(sudo_wrapper)
    end
  end
end
```
这样确实可以规避嵌套的sudo的调用。但相互并行的方法就不行了。
不过，如果需要调用sudo的地方不多，还是可以的。

而我们一开始就往尽可能减少创建进程。并且只将sudo调用的方法限于FileUtility这个类里面.
```ruby
def get_sudo_wrapper
  @sudo ||= lambda do
    require "sudo"
    sudo = Sudo::Wrapper.new(load_gems: false)
    sudo.start!
    sudo
  end.call
end
private :get_sudo_wrapper

#需要使用的地方:
sudo = get_sudo_wrapper
sudo[FileUtils].ln_sf(source, target)

```
这个方法确实只创建了一个进程，但这个进程会一直存在。就算调用的起始进程退出了，它还在。

后面就想着开启一个async的异步事件来手工调用 sudo.stop.

```ruby
def finish_sudo()
  return if @sudo.nil?
  sudo_instance = @sudo
  @sudo = nil
  sudo_instance.stop!
end
```

很古怪的是，进程杀不了。跟踪发现确实调用了 Sudo::System.kill h[:pid]
```ruby
def kill(pid)
  if pid and Process.exists? pid
    system "sudo kill     #{pid}" or
    system "sudo kill -9  #{pid}" or
    raise ProcessStillExists, "Couldn't kill sudo process (PID=#{pid})"
  end
end
```
传入的进程pid也是对的。就是杀不了.

查看sudo-0.2.0/libexec/server.rb的实现，对照drb的文档.
```ruby
DRb.start_service(uri, Sudo::Proxy.new)
FileUtils.chown owner, 0, socket
FileUtils.chmod 0600, socket
DRb.thread.join
```
Drb.start_service会开启server,并且会有线程一直在运行.
比较优雅的退出版本，就是Drb.stop_service.
第一想到的，就是hook一个方法，发送一个命令，让它里面调用Drb.stop_service.
后面想想,drb本来就是远程包装对象，然后将对象的方法调用，远程代理到相同的对象(使用marshalled)

Any type of object can be passed as an argument to a dRuby call or returned as its return value. By default, such objects are dumped or marshalled at the local end, then loaded or unmarshalled at the remote end. The remote end therefore receives a copy of the local object, not a distributed reference to it; methods invoked upon this copy are executed entirely in the remote process, not passed on to the local original. This has semantics similar to pass-by-value.

那我们直接用 sudo[Drb].stop_service如何?

```ruby
      require "monitor"
      extend MonitorMixin
### synchronize support reentrance.

      def watch_sudo_wrapper(delay_ms: nil, &block)
        delay_ms = delay_ms.to_i
        if delay_ms < 200
          delay_ms = 500
          delay_ms = 3000 if Utility::AppEnv.production?
        end
        Core::AsyncScheduler.instance.fast_schedule_async(delay_ms: delay_ms) do
          if block.nil?
            Utility::FileUtility.finish_sudo()
          else
            block.call()
          end
        end
      end

      private :watch_sudo_wrapper


      def get_sudo_wrapper
        self.synchronize do
          @sudo ||= lambda do
            require "sudo"
            sudo = Sudo::Wrapper.new(load_gems: false)
            sudo.start!
            watch_sudo_wrapper
            sudo
          end.call
        end
      end

      private :get_sudo_wrapper

      def finish_sudo()
        self.synchronize do
          return if @sudo.nil?
          sudo_instance = @sudo
          @sudo = nil
          sudo_instance[DRb].stop_service
          ensure_stop = lambda do
            sudo_instance.stop!
          end
          watch_sudo_wrapper(delay_ms: 2000, &ensure_stop)
        end
      end

```

经过测试，发现问题完美解决。基本不需要调用sudo_instance.stop! 进程就已经优雅的关闭了。
我们加上sudo_instance.stop!只是为了failback.

由于我们sudo只在很小的方法里面使用，基本上不会耗时多少，基本上1秒之内就可以完成。
所以，一旦我们开始调用finish_sudo的时候，恰好其他线程获刚调用get_sudo_wrapper也没有关系，
因为DRb.stop_service会等待正在操作的业务完成。

至此，我们将FileUtility将很多文件操作，都加入了一个参数 use_sudo:false的默认参数。
在操作文件很方便，没有感觉到使用sudo有什么不方便.只是多个标记而已.


将FileUtility的实现贴在下面.有需要的就直接使用吧，操作起来还是挺方便的.

```ruby
require "monitor"
module Utility
  module FileUtility
    extend MonitorMixin

    def file?(name, use_sudo: false)
      return true if cache_file?(filename: name)
      unless use_sudo
        File.file?(name)
      else
        sudo = get_sudo_wrapper
        sudo[File].file?(name)
      end
    end

    def directory?(name, use_sudo: false)
      unless use_sudo
        File.directory?(name)
      else
        sudo = get_sudo_wrapper
        sudo[File].directory?(name)
      end
    end

    def exist?(name, use_sudo: false)
      return true if cache_file?(filename: name)
      unless use_sudo
        File.exist?(name)
      else
        sudo = get_sudo_wrapper
        sudo[File].exist?(name)
      end
    end

    def size(name, use_sudo: false)
      unless use_sudo
        File.size(name)
      else
        sudo = get_sudo_wrapper
        sudo[File].size(name)
      end
    end

    def pre_outdir(dir:, keep_origin: true, use_sudo: false)
      if exist?(dir, use_sudo: use_sudo) && !keep_origin
        begin
          FileUtils.rmtree(dir)
        rescue Errno::EACCES
          raise unless use_sudo
          sudo = get_sudo_wrapper
          sudo[FileUtils].rmtree(dir)
        end
      end
      mkdir_path(dir: dir, use_sudo: use_sudo)
    end

    def rm_empty_dir(dir:, use_sudo: false)
      return unless directory?(dir, use_sudo: use_sudo)
      while true
        #if Dir.entries(dir).size <= 2
        begin
          Dir.rmdir(dir)
        rescue Errno::EACCES
          raise unless use_sudo
          begin
            sudo = get_sudo_wrapper
            sudo[Dir].rmdir(dir)
          rescue SystemCallError
            break
          end
        rescue SystemCallError
          break
        end
        dir = File.dirname(dir)
        break if dir.empty?
      end
    end

    def mkdir_path(dir:, use_sudo: false, fetch_opt: nil)
      begin
        ret = FileUtils.mkpath(dir)
        if fetch_opt.is_a?(Hash)
          fetch_opt[:use_sudo] = false
        end
        ret
      rescue Errno::EACCES
        raise unless use_sudo
        sudo = get_sudo_wrapper
        sudo[FileUtils].mkpath(dir)
      end
    end

    def pre_outfile(out_file:, keep_origin: true, use_sudo: false, fetch_opt: nil)
      out_dirname = File.dirname(out_file)
      mkdir_path(dir: out_dirname, use_sudo: use_sudo, fetch_opt: fetch_opt)
      unless keep_origin
        if file?(out_file, use_sudo: use_sudo)
          rm_file(file: out_file, use_sudo: use_sudo)
        elsif directory?(out_file, use_sudo: use_sudo)
          remove_dir(dir: out_file, use_sudo: use_sudo)
        end
      end
    end

    def copy_file(source:, target:, use_sudo: false)
      fetch_opt = nil
      if use_sudo
        fetch_opt = {}
      end
      pre_outfile(out_file: target, use_sudo: use_sudo, fetch_opt: fetch_opt)
      begin
        ::FileUtils.cp(source, target)
      rescue Errno::EACCES
        raise unless use_sudo
        if fetch_opt[:use_sudo]
          sudo = get_sudo_wrapper
          sudo[FileUtils].cp(source, target)
        else
          ### source is sudo, but target is not sudo.
          content = read_file_content(filename: source, use_sudo: use_sudo)
          write_file(target: target, content: content)
        end
      end
    end

    alias :cp :copy_file

    def copy_to_dir(source:, target:, use_sudo: false)
      if file?(source, use_sudo: use_sudo)
        target_file = File.join(target, File.basename(source))
        return copy_file(source: source, target: target_file, use_sudo: use_sudo)
      end
      copy_dir(source: source, target: target, use_sudo: use_sudo)
    end

    ##拷贝目录.
    def copy_dir(source:, target:, use_sudo: false)
      pre_outdir(dir: target, use_sudo: use_sudo)
      begin
        ::FileUtils.copy_entry(source, target)
      rescue Errno::EACCES
        sudo = get_sudo_wrapper
        sudo[FileUtils].copy_entry(source, target)
      end
    end

    ###两个都是文件名.
    def move_file(source:, target:, use_sudo: false)
      pre_outfile(out_file: target, use_sudo: use_sudo)
      begin
        ::FileUtils.mv(source, target, force: true)
      rescue Errno::EACCES
        raise unless use_sudo
        sudo = get_sudo_wrapper
        sudo[FileUtils].mv(source, target) #force: true
      end
    end

    def move_to_dir(source:, dir:, use_sudo: false)
      pre_outdir(dir: dir, use_sudo: use_sudo)
      begin
        ::FileUtils.mv(source, target, force: true)
      rescue Errno::EACCES
        raise unless use_sudo
        sudo = get_sudo_wrapper
        sudo[FileUtils].mv(source, target) #, force: true
      end
    end

    def remove(target:, use_sudo: false)
      if directory?(target, use_sudo: use_sudo)
        #::FileUtils.rm_rf(target)
        remove_dir(dir: target, use_sudo: use_sudo)
      else
        rm_file(file: target, use_sudo: use_sudo)
      end
    end

    def remove_dir(dir:, use_sudo: false)
      FileUtils.rmtree(dir)
      if use_sudo
        if directory?(dir, use_sudo: use_sudo)
          raise Errno::EACCES.new("coundn't remove dir #{dir}")
        end
      end
    rescue Errno::EACCES
      raise unless use_sudo
      if keep_dirs.include?(dir)
        raise "could not remove reserved dir. #{dir}"
      end
      sudo = get_sudo_wrapper
      sudo[FileUtils].rmtree(dir)
    end

    def keep_dirs
      @_arr_keep_dirs ||= ["/", "/root", "/lib", "/lib64", "/lib32",
                           "/bin", "/etc", "/home",
                           "/var", "/sbin", "/boot", "/usr", ""]
    end

    private :keep_dirs

    def rm_file(file:, use_sudo: false)
      FileUtils.rm_f(file)
      if use_sudo
        if file?(file, use_sudo: use_sudo)
          raise Errno::EACCES.new("coundn't remove file #{file}")
        end
      end
    rescue Errno::EACCES
      raise unless use_sudo
      sudo = get_sudo_wrapper
      sudo[FileUtils].rm_f(file)
    end

    def write_file(target:, content:, use_sudo: false)
      pre_outfile(out_file: target, use_sudo: use_sudo)
      begin
        File.open(target, "wb") do |_file|
          _file.write(content)
        end
      rescue Errno::EACCES
        raise unless use_sudo
        ###
        require "tempfile"
        tmp_file = Tempfile.new()
        tmp_file.write(content)
        tmp_file.rewind
        tmp_file.close
        file_path = tmp_file.path
        sudo = get_sudo_wrapper
        sudo[::FileUtils].mv(file_path, target)
        tmp_file.unlink
      end
    end

    def read_file_content(filename:, dir: nil, default: "", use_sudo: false)
      unless dir.nil?
        filename = File.expand_path(filename, dir)
      end
      if cache_file?(filename: filename)
        return get_file_cached_content(filename: filename, default: default)
      end
      if file?(filename, use_sudo: use_sudo)
        begin
          return File.read(filename)
        rescue Errno::EACCES
          raise unless use_sudo
          sudo = get_sudo_wrapper
          return sudo[File].read(filename)
        end
      end
      default
    end

    ### just for rspec.
    def cache_file_content(abs_filename:, content:)
      cache_map = cache_file_map
      if content.nil?
        cache_map.delete(abs_filename)
      else
        cache_map[abs_filename] = content.to_s
      end
    end

    ##将source文件link到target.
    def link_file(source:, target:, use_sudo: false)
      unless file?(source, use_sudo: use_sudo)
        raise "source #{source} is not file"
      end
      if directory?(target, use_sudo: use_sudo)
        raise "source #{target} is a directory"
      end
      pre_outfile(out_file: target, keep_origin: false, use_sudo: use_sudo)
      FileUtils.ln_s(source, target, force: true)
    rescue Errno::EACCES
      raise unless use_sudo
      sudo = get_sudo_wrapper
      sudo[FileUtils].ln_sf(source, target)
    end

    def link_dir(source:, target:, use_sudo: false)
      unless directory?(source, use_sudo: use_sudo)
        raise "source #{source} is not directory"
      end
      pre_outfile(out_file: target, keep_origin: false, use_sudo: use_sudo)
      FileUtils.ln_s(source, target, force: true)
    rescue Errno::EACCES
      raise unless use_sudo
      sudo = get_sudo_wrapper
      sudo[FileUtils].ln_sf(source, target)
    end

    def watch_sudo_wrapper(delay_ms: nil, &block)
      delay_ms = delay_ms.to_i
      if delay_ms < 200
        delay_ms = 500
        delay_ms = 1500 if Utility::AppEnv.production?
      end
      Core::AsyncScheduler.instance.fast_schedule_async(delay_ms: delay_ms) do
        if block.nil?
          Utility::FileUtility.finish_sudo()
        else
          block.call()
        end
      end
    end

    private :watch_sudo_wrapper

    def get_sudo_wrapper
      self.synchronize do
        @sudo ||= lambda do
          require "sudo"
          sudo = Sudo::Wrapper.new(load_gems: false)
          sudo.start!
          watch_sudo_wrapper
          sudo
        end.call
      end
    end

    private :get_sudo_wrapper

    def finish_sudo()
      self.synchronize do
        return if @sudo.nil?
        sudo_instance = @sudo
        @sudo = nil
        sudo_instance[DRb].stop_service
        ensure_stop = lambda do
          sudo_instance.stop!
        end
        watch_sudo_wrapper(delay_ms: 2000, &ensure_stop)
      end
    end

    private

    def cache_file_map
      @cache_map ||= {}
    end

    def cache_file?(filename:)
      cache_file_map.key?(filename)
    end

    def get_file_cached_content(filename:, default: nil)
      cache_file_map.fetch(filename, default)
    end
  end
end


```

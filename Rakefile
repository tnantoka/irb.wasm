$LOAD_PATH << File.join(File.dirname(__FILE__), "vendor", "deps", "ruby.wasm", "lib")

require "rake/tasklib"
retry_count = 0
begin
  require "ruby_wasm/build_system"
  require "ruby_wasm/rake_task"
rescue LoadError => e
  if retry_count == 0
    sh "git submodule update --init"
    retry_count += 1
    retry
  else
    raise e
  end
end

Dir.glob("tasks/**.rake").each { |f| import f }

FULL_EXTS = "bigdecimal,cgi/escape,continuation,coverage,date,dbm,digest/bubblebabble,digest,digest/md5,digest/rmd160,digest/sha1,digest/sha2,etc,fcntl,fiber,gdbm,json,json/generator,json/parser,nkf,objspace,pathname,psych,racc/cparse,rbconfig/sizeof,ripper,stringio,strscan,monitor,zlib"

LIB_ROOT = File.expand_path("../vendor/deps/ruby.wasm", __FILE__)
BUILD_DIR = File.join(Dir.pwd, "build")

options = {
  target: "wasm32-unknown-wasi",
  src: { name: "head", type: "github", repo: "ruby/ruby", rev: "817193104dad2eb3f7b9593e2164cc88b3a54887", patches: [] },
  default_exts: FULL_EXTS,
  build_dir: BUILD_DIR,
}

channel = "head-wasm32-unknown-wasi-full-js-irb"

build_task = RubyWasm::BuildTask.new(channel, **options) do |t|
  t.crossruby.user_exts = [
    RubyWasm::CrossRubyExtProduct.new(File.join(LIB_ROOT, "ext", "js"), t.toolchain),
    RubyWasm::CrossRubyExtProduct.new(File.join(LIB_ROOT, "ext", "witapi"), t.toolchain),
  ]
  t.crossruby.debugflags = %w[-g0]
  t.crossruby.wasmoptflags = %w[-O3]
  t.crossruby.ldflags = %w[
    -Xlinker
    --stack-first
    -Xlinker
    -z
    -Xlinker
    stack-size=16777216
  ]
end

task :cache_key do
  puts build_task.hexdigest
end

wasi_vfs = build_task.wasi_vfs

RUBY_ROOT = File.join("rubies", channel)

task :default => "static/irb.wasm"

desc "Build irb.wasm"
file "static/irb.wasm" => [] do
  wasi_vfs.install_cli
  unless File.exist?(RUBY_ROOT)
    Rake::Task[channel].invoke
  end
  require "tmpdir"
  Dir.mktmpdir do |dir|
    tmpruby = File.join(dir, "ruby")
    cp_r RUBY_ROOT, tmpruby
    rm_rf File.join(tmpruby, "usr", "local", "include")
    rm_f File.join(tmpruby, "usr", "local", "lib", "libruby-static.a")
    ruby_wasm = File.join(dir, "ruby.wasm")
    mv File.join(tmpruby, "usr", "local", "bin", "ruby"), ruby_wasm
    sh "#{wasi_vfs.cli_bin_path} pack #{ruby_wasm} --mapdir /usr::#{File.join(tmpruby, "usr")} --mapdir /gems::./fake-gems -o static/irb.wasm"
  end
end

desc "Deep clean build artifacts"
task :deep_clean => :clean do
  Rake::Task[channel + ":clean"].invoke
end

desc "Clean build artifacts"
task :clean do
  rm_f "static/irb.wasm"
end

desc "Start local parcel server"
task :parcel do
  sh "npx parcel ./src/index.html"
end

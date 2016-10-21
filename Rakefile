require 'rubygems'
require 'bundler/setup'
require 'git'
require 'open3'
require 'fileutils'

task default: %w[package_concat]

GOPATH = ENV['GOPATH']
TRAVIS_GOPATH_CACHE = "#{ENV['HOME']}/gocache"
CONCAT_URL = 'https://github.com/mediachain/concat'
CONCAT_WORKING_DIR = "#{GOPATH}/src/github.com/mediachain/concat"

def travis_os
  name = ENV['TRAVIS_OS_NAME']
  if name == nil
    return :not_on_travis
  end
  return name.to_sym
end

def on_travis?
  return travis_os != :not_on_travis
end

def checkout_project(origin_url, working_dir, ref_str)
  if Dir.exist?(working_dir)
    g = Git.open(working_dir)
  else
    g = Git.clone(origin_url, working_dir)
  end

  g.fetch('origin', {tags: true})
  begin
    ref = g.revparse(ref_str)
  rescue
    ref_str = "origin/#{ref_str}"
    ref = g.revparse(ref_str)
  end
  g.checkout(ref)
end

def run_cmd(cmd)
  Open3.popen3(cmd) { |input,output,error,wait_thread|
    output.each do |line|
      puts line
    end

    status = wait_thread.value
    unless status.to_i == 0
      STDERR.puts "Error running command #{cmd}"
      IO.copy_stream(error, STDERR)
      exit status
    end
  }
end

def cgo_env
  if travis_os == :linux
    return 'CC="gcc-4.8" CXXFLAGS="-stdlib=libc++" LDFLAGS="-stdlib=libc++ -std=c++11 -lrt -Wl,--no-as-needed"'
  end
  return ""
end

def rsync(src, dest)
  run_cmd("rsync #{src} #{dest}")
end

task :checkout_concat do
  concat_ref = ENV['concat_ref'] || 'master'
  puts "concat: checking out ref: #{concat_ref}"
  checkout_project(CONCAT_URL, CONCAT_WORKING_DIR, concat_ref)
end

task concat_deps: [:checkout_concat] do
  Dir.chdir(CONCAT_WORKING_DIR) do
    puts "concat: installing dependencies"
    run_cmd("env #{cgo_env} ./setup.sh")
  end
end

task build_concat: [:restore_go_cache, :concat_deps] do
  Dir.chdir(CONCAT_WORKING_DIR) do
    puts "concat: building project"
    run_cmd('./install.sh')
  end
end

task create_go_cache: [:concat_deps] do
  if !on_travis?
    next
  end

  unless Dir.exist? TRAVIS_GOPATH_CACHE
    Dir.mkdir TRAVIS_GOPATH_CACHE
  end

  puts "copying go cache from #{TRAVIS_GOPATH_CACHE} to #{GOPATH}"
  rsync("#{GOPATH}/src", "#{TRAVIS_GOPATH_CACHE}/src")
  rsync("#{GOPATH}/pkg", "#{TRAVIS_GOPATH_CACHE}/pkg")
end

task :restore_go_cache do
  if !on_travis?
    next
  end

  unless Dir.exist? TRAVIS_GOPATH_CACHE
    puts "travis go cache does not exist at #{TRAVIS_GOPATH_CACHE}, skipping cache restore"
    next
  end

  rsync("#{TRAVIS_GOPATH_CACHE}/src", "#{GOPATH}/src")
  rsync("#{TRAVIS_GOPATH_CACHE}/pkg", "#{GOPATH}/pkg")
end

task package_concat: [:build_concat, :create_go_cache] do
  tarball = "#{Dir.pwd}/concat.tgz"
  Dir.chdir("#{GOPATH}/bin") do
    run_cmd("tar cfz #{tarball} mcnode mcdir")
  end
  puts "concat: created #{tarball}"
end

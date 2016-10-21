require 'rubygems'
require 'bundler/setup'
require 'git'
require 'open3'

task default: %w[package_concat]

GOPATH = ENV['GOPATH']
CONCAT_URL = 'https://github.com/mediachain/concat'
CONCAT_WORKING_DIR = "#{GOPATH}/src/github.com/mediachain/concat"

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

task :checkout_concat do
  concat_ref = ENV['concat_ref'] || 'master'
  puts "concat: checking out ref: #{concat_ref}"
  checkout_project(CONCAT_URL, CONCAT_WORKING_DIR, concat_ref)
end

task concat_deps: [:checkout_concat] do
  Dir.chdir(CONCAT_WORKING_DIR) do
    puts "concat: installing dependencies"
    run_cmd('./setup.sh')
  end
end

task build_concat: [:concat_deps] do
  Dir.chdir(CONCAT_WORKING_DIR) do
    puts "concat: building project"
    run_cmd('./install.sh')
  end
end


task package_concat: [:build_concat] do
  tarball = "#{Dir.pwd}/concat.tgz"
  Dir.chdir("#{GOPATH}/bin") do
    run_cmd("tar cfz #{tarball} mcnode mcdir")
  end
  puts "concat: created #{tarball}"
end

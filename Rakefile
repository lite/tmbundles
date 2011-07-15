require 'rake'
require 'erb'

$my_verbose = 1
$my_verbose = false if $my_verbose == 0

desc "update then install"
task :default => [:update, :install] do
end

desc "Update the files"
task :update => [:pull, :upgrade] do
end

desc "Pull new changes, via `git pull`"
task :pull do
  puts "Pulling.." if $my_verbose
  system %Q{git pull} or raise "Git pull failed."
end

desc "Upgrade submodules to current master"
task :upgrade do
  ignore_modified = true
  system %Q{ git diff --cached --exit-code > /dev/null } or raise "The git index is not clean."

  submodules = {}
  get_submodule_status.each do |path, sm|
    if ignore_modified && sm["state"] == "+"
      puts "Skipping modified submodule #{path}."
      next
    end
    if sm["state"] == "-"
      puts "Skipping uninitialized submodule #{path}."
      next
    end
    submodules[path] = sm
  end

  begin
  submodules.each do |path,sm|
    puts "Upgrading #{path}.." if $my_verbose

    # puts "Fetching all remotes" if $my_verbose
    output = %x[cd #{path} && git fetch --all]

    sm_url = %x[git config --get submodule.#{path}.url].chomp
    puts "Fetching #{path} from #{sm_url}" if $my_verbose
    output = %x[cd #{path} && git fetch #{sm_url} 2>&1]
    puts output if $my_verbose and $my_verbose > 1
    if not $?.success?
      raise "Fetching failed: " + output
    end

    # Check that current commit is ancestor of FETCH_HEAD
    # sm_commit = %x[cd #{path} && git rev-parse #{sm["commit"]}].chomp
    sm_commit = sm['commit']
    merge_base = %x[cd #{path} && git merge-base #{sm_commit} FETCH_HEAD].chomp
    if sm_commit != merge_base
      # puts "Skipping #{path}: Current commit does not appear to be ancestor of FETCH_HEAD."
      # puts "Info: sm_commit: #{sm_commit}, merge_base: #{merge_base}"
      # next
      output = %x[cd #{path} && git merge FETCH_HEAD]
      puts "Merged FETCH_HEAD:\n" + output
    end

    output = %x[cd #{path} && git merge --ff-only FETCH_HEAD 2>&1]
    if ! output.split("\n")[-1] =~ /^Already up-to-date.( Yeeah!)?/
      puts output
    else
      puts output if $my_verbose and $my_verbose > 1
      # TODO: pull result
    end
    if not $?.success?
      raise "Merging FETCH_HEAD failed: " + output
    end
    next

    # 1. get available remotes
    # 2. find newest one, via
    #    git --describe always
    #    git branch (-a) --contains $DESC
    #    get available master branches via gb -r
    # remotes = %x[git --git-dir '#{path}/.git' remote]
    # if not $?.success?
    #   raise "Pulling failed: " + output
    # end
    #
    output = output.split("\n")
    # Output important lines
    puts output.select{|x| x=~/^Your branch is/}
  end
  rescue Exception => exc
    puts "Exception: " + exc.message
    puts exc.backtrace.join("\n")
  ensure
    Rake::Task[:commitsubmodules].invoke(submodules)
  end
end

desc "install the tmbundles into '~/Library/Application Support/TextMate/Bundles' directory"
task :install do
  base = "~/Library/Application Support/TextMate/Bundles"
  if not base
    system %Q{mkdir -p "#{bash}"}
  end

  Dir['*'].each do |file|
    next if %w[Rakefile README.md].include? file

    puts "linking #{file}"

    link_file(File.expand_path(file), File.expand_path(File.join(base, file)))
  end
  
  # reload textmate bundles
  system %x[osascript -e 'tell application \"TextMate\" to reload bundles'] 
  
end

def link_file(file, target)
  system %Q{ln -sfn "#{file}" "#{target}"} 
end

def get_submodule_status(sm_args='')
  # return { 'vim/bundle/solarized' => {'state'=>' '} }
  puts "Getting submodules status.." if $my_verbose
  status = %x[ git submodule status #{sm_args} ] or raise "Getting submodule status failed"
  r = {}
  status.split("\n").each do |line|
    if not line =~ /^([ +-])(\w{40}) (.*?)(?: \(.*\))?$/
      raise "Found unexpected submodule line: #{line}"
    end
    path = $3
    next if ! path
    r[path] = {"state" => $1, "commit" => $2}
  end
  return r
end

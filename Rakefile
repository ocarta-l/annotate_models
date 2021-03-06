def exit_exception(e)
  $stderr.puts e.message
  exit e.status_code
end

# NOTE: this causes annoying psych warnings under Ruby 1.9.2-p180; to fix, upgrade to 1.9.3
begin
  require 'bundler'
  Bundler.setup(:default, :development)
rescue Bundler::BundlerError => e
  $stderr.puts 'Run `bundle install` to install missing gems'
  exit_exception(e)
end

using_dsl = false
begin
  require 'rake/dsl_definition'
  using_dsl = true
rescue Exception => e
  # We might just be on an old version of Rake...
  exit_exception(e)
end
require 'rake'
include Rake::DSL if using_dsl

require './lib/annotate'
require 'mg'
begin
  MG.new('annotate.gemspec')
rescue Exception
  $stderr.puts("WARNING: Couldn't read gemspec.  As such, a number of tasks may be unavailable to you until you run 'rake gem:gemspec' to correct the issue.")
  # Gemspec is probably in a broken state, so let's give ourselves a chance to
  # build a new one...
end
DEVELOPMENT_GROUPS = [:development, :test].freeze
RUNTIME_GROUPS = Bundler.definition.groups - DEVELOPMENT_GROUPS
namespace :gem do
  task :gemspec do
    spec = Gem::Specification.new do |gem|
      # See http://docs.rubygems.org/read/chapter/20
      # for more options.
      gem.version = Annotate.version
      gem.name = 'annotate'
      gem.homepage = 'http://github.com/ctran/annotate_models'
      gem.rubyforge_project = 'annotate'
      gem.license = 'Ruby'
      gem.summary = 'Annotates Rails Models, routes, fixtures, and others based on the database schema.'
      gem.description = 'Annotates Rails/ActiveRecord Models, routes, fixtures, and others based on the database schema.'
      gem.email = ['alex@stinky.com', 'cuong@gmail.com', 'x@nofxx.com', 'turadg@aleahmad.net', 'jon@cloudability.com']
      gem.authors = ['Alex Chaffee', 'Cuong Tran', 'Marcos Piccinini', 'Turadg Aleahmad', 'Jon Frisby']
      gem.require_paths = ['lib']
      # gem.rdoc_options = ["--charset=UTF-8"]
      # gem.required_ruby_version = "> 1.9.2"

      Bundler.load.dependencies_for(*RUNTIME_GROUPS).each do |dep|
        runtime_resolved = Bundler.definition.specs_for(RUNTIME_GROUPS).find { |spec| spec.name == dep.name }
        unless runtime_resolved.nil?
          gem.add_dependency(dep.name, dep.requirement)
        end
      end

      gem.executables = `git ls-files -- bin/*`.split("\n").map { |f| File.basename(f) }
      gem.extra_rdoc_files = ['README.md', 'CHANGELOG.md', 'TODO.md']

      gem.files = `git ls-files -- .`.split("\n").reject do |fn|
        fn =~ /^Gemfile.*/ ||
          fn =~ /^Rakefile/ ||
          fn =~ /^\.rvmrc/ ||
          fn =~ /^\.gitignore/ ||
          fn =~ /^\.rspec/ ||
          fn =~ /^\.document/ ||
          fn =~ /^\.yardopts/ ||
          fn =~ /^pkg/ ||
          fn =~ /^spec/ ||
          fn =~ /^doc/ ||
          fn =~ /^vendor\/cache/
      end.sort
    end
    File.open('annotate.gemspec', 'wb') do |fh|
      fh.write("# This file is auto-generated!\n")
      fh.write("# DO NOT EDIT THIS FILE DIRECTLY!\n")
      fh.write("# Instead, edit the Rakefile and run 'rake gems:gemspec'.")
      fh.write(spec.to_ruby)
    end
  end
end

namespace :jeweler do
  task :clobber do
    FileUtils.rm_f('pkg')
  end
end
task clobber: :'jeweler:clobber'

require 'rspec/core/rake_task' # RSpec 2.0
RSpec::Core::RakeTask.new(:spec) do |t|
  t.pattern = ['spec/*_spec.rb', 'spec/**/*_spec.rb']
  t.rspec_opts = ['--backtrace', '--format d']
end

# Placeholder for running bin/* in development...
task :environment

task :integration_environment do
  require './spec/spec_helper'
end

namespace :gemsets do
  desc "Completely empty any gemsets used by scenarios, so they'll be perfectly clean on the next run."
  task empty: [:integration_environment] do
    Annotate::Integration::SCENARIOS.each do |test_rig, _base_dir, _test_name|
      Annotate::Integration.empty_gemset(test_rig)
    end
  end
end
task clobber: :'gemsets:empty'

namespace :integration do
  desc "Remove any cruft generated by manual debugging runs which is .gitignore'd."
  task clean: :integration_environment do
    Annotate::Integration.nuke_all_cruft
  end

  desc 'Reset any changed files, and remove any untracked files in spec/integration/*/, plus run integration:clean.'
  task clobber: [:integration_environment, :'integration:clean'] do
    Annotate::Integration.reset_dirty_files
    Annotate::Integration.clear_untracked_files
  end

  task symlink: [:integration_environment] do
    require 'digest/md5'

    integration_dir = File.expand_path(File.join(File.dirname(__FILE__), 'spec', 'integration'))
    # fixture_dir = File.expand_path(File.join(File.dirname(__FILE__), 'spec', 'fixtures'))
    target_dir = File.expand_path(ENV['TARGET']) if ENV['TARGET']
    raise 'Must specify TARGET=x, where x is an integration test scenario!' unless target_dir && Dir.exist?(target_dir)
    raise 'TARGET directory must be within spec/integration/!' unless target_dir.start_with?(integration_dir)
    candidates = {}
    FileList[
      "#{target_dir}/.rvmrc",
      "#{target_dir}/**/*"
    ].select { |fname| !(File.symlink?(fname) || File.directory?(fname)) }
      .map { |fname| fname.sub(integration_dir, '') }
      .reject do |fname|
        fname =~ /\/\.gitkeep$/ ||
          fname =~ /\/app\/models\// ||
          fname =~ /\/routes\.rb$/ ||
          fname =~ /\/fixtures\// ||
          fname =~ /\/factories\// ||
          fname =~ /\.sqlite3$/ ||
          (fname =~ /\/test\// && fname !~ /_helper\.rb$/) ||
          (fname =~ /\/spec\// && fname !~ /_helper\.rb$/)
      end
      .map { |fname| "#{integration_dir}#{fname}" }
      .each do |fname|
        digest = Digest::MD5.hexdigest(File.read(fname))
        candidates[digest] ||= []
        candidates[digest] << fname
      end
    fixtures = {}
    FileList['spec/fixtures/**/*'].each do |fname|
      fixtures[Digest::MD5.hexdigest(File.read(fname))] = File.expand_path(fname)
    end

    candidates.each_key do |digest|
      next unless fixtures.key?(digest)
      candidates[digest].each do |fname|
        # Double-check contents in case of hash collision...
        next unless FileUtils.identical?(fname, fixtures[digest])
        destination_dir = Pathname.new(File.dirname(fname))
        relative_target = Pathname.new(fixtures[digest]).relative_path_from(destination_dir)
        Dir.chdir(destination_dir) do
          sh('ln', '-sfn', relative_target.to_s, File.basename(fname))
        end
      end
    end
  end
end
task clobber: :'integration:clobber'

require 'yard'
YARD::Rake::YardocTask.new do |t|
  # t.files   = ['features/**/*.feature', 'features/**/*.rb', 'lib/**/*.rb']
  # t.options = ['--any', '--extra', '--opts'] # optional
end

namespace :yard do
  task :clobber do
    FileUtils.rm_f('.yardoc')
    FileUtils.rm_f('doc')
  end
end
task clobber: :'yard:clobber'

namespace :rubinius do
  task :clobber do
    FileList['**/*.rbc'].each { |fname| FileUtils.rm_f(fname) }
    FileList['.rbx/**/*'].each { |fname| FileUtils.rm_f(fname) }
  end
end
task clobber: :'rubinius:clobber'

# want other tests/tasks run by default? Add them to the list
task default: [:spec]

    require 'rubygems'
    require 'rake'
    require 'rdoc'
    require 'date'
    require 'yaml'
    require 'tmpdir'
    require 'jekyll'

    desc "Generate blog files"
    task :generate do
      Jekyll::Site.new(Jekyll.configuration({
        "source"      => ".",
        "destination" => "_site"
      })).process
    end


    desc "Generate and publish blog to gh-pages"
    task :publish => [:generate] do
      puts "\n## Deleting gh-pages branch"
      status = system("git branch -D gh-pages")
      puts status ? "Success" : "Failed"
      puts "\n## Creating new gh-pages branch and switching to it"
      status = system("git checkout -b gh-pages")
      puts status ? "Success" : "Failed"
      puts "\n## Forcing the _site subdirectory to be project root"
      status = system("git filter-branch --subdirectory-filter _site/ -f")
      puts status ? "Success" : "Failed"
      puts "\n## Pushing gh-pages branches to origin"
      status = system("git push origin gh-pages")
      puts status ? "Success" : "Failed"
      puts "\n## Switching back to master branch"
      status = system("git checkout master")
      puts status ? "Success" : "Failed"
    end

task :default =>:publish

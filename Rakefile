#    Copyright (C) 2010 Alexandre Berman, Lazybear Consulting (sashka@lazybear.net)
#
#    This program is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 3 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License
#    along with this program.  If not, see <http://www.gnu.org/licenses/>.
#
#    ::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
#
# -- usage: rake help

require "timeout"
require "fileutils"

# -- global vars
task :default => [:run]
@test_dir              = "tests"
@excludes              = ".svn"
@tests                 = []
@tests_retried_counter = 0
@executed_tests        = 0
@reports_dir           = ENV['HOME'] + "/rake_reports" # -- default
@reports_dir           = ENV['REPORTS_DIR'] if ENV['REPORTS_DIR'] != nil
#
# -- the following vars control the behavior of running tests
@test_data = {
   'output_on'                => false,
   'test_retry'               => true,
   'test_exit_status_passed'  => "PASSED",
   'test_exit_status_failed'  => "FAILED",
   'test_exit_status_skipped' => "SKIPPED",
   'xml_report_class_name'    => "qa.tests",
   'xml_report_file_name'     => "TESTS-TestSuites.xml",
   'reports_dir'              => @reports_dir
}

# -- usage
desc "-- usage"
task :help do
    puts "\n-- usage: \n\n"
    puts "   rake help                                     : print this message"
    puts "   rake                                          : this will by default run :run task, which runs all tests"
    puts "   rake run KEYWORDS=<keyword1,keyword2>         : this will run tests based on keyword"
    puts "   rake print_human                              : this will print descriptions of your tests"
    puts "   rake print_human KEYWORDS=<keyword1,keyword2> : same as above, but only for tests corresponding to KEYWORDS"
    puts "   rake REPORTS_DIR=</path/to/reports>           : this will set default reports dir and run all tests\n\n\n\n"
    puts "   Eg:\n\n   rake KEYWORDS=<keyword1, keyword2> REPORTS_DIR=/somewhere/path\n\n\n"
    puts "   Sample comments in your tests (eg: tests/some_test.rb):\n\n"
    puts "   # @author Alexandre Berman"
    puts "   # @executeArgs"
    puts "   # @keywords acceptance"
    puts "   # @description some interesting test\n\n"
    puts "   Note 1:\n\n   Your test must end with 'test.rb' - otherwise Rake won't be able to find it, eg:\n"
    puts "   tests/some_new_test.rb\n\n"
    puts "   Note 2:\n\n   Your test must define at least one keyword.\n\n"
end

# -- prepare reports_dir
def prepare_reports_dir
   FileUtils.rm_r(@test_data['reports_dir']) if File.directory?(@test_data['reports_dir'])
   FileUtils.mkdir_p(@test_data['reports_dir'])
end

# -- our own each method yielding each test in the @tests array
def each
   @tests.each { |t| yield t }
end

# -- filtering by keywords
def filter_by_keywords
   # -- do we have keywords set ?
   if (ENV['KEYWORDS'] != nil and ENV['KEYWORDS'].length > 0)
      tmp_tests = []
      # -- loop through all keywords
      ENV['KEYWORDS'].gsub(/,/, ' ').split.each { |keyword|
         # -- loop through all tests
         @tests.each { |t|
            # -- loop through all keywords for a given test
            t.keywords.each { |k|
               if k == keyword
                  tmp_tests << t
               end
            }
         }
      }

      # -- in case only a negative keyword was given, let's fill up tmp_tests array here (ie: if it is empty now):
      tmp_tests = @tests.uniq if tmp_tests.length < 1 and /!/.match(ENV['KEYWORDS'])

      # -- check for a negative keyword
      if /!/.match(ENV['KEYWORDS'])
         ENV['KEYWORDS'].gsub(/,/, ' ').split.each { |keyword|
            if /!/.match(keyword)
               keyword.gsub!(/!/, '')
               # -- loop through all tests
               @tests.each { |t|
                  # -- loop through all keywords for a given test
                  t.keywords.each { |k|
                     # -- if keyword matches with negative keyword, remove this test from the array
                     if k == keyword
                        tmp_tests.delete(t)
                     end
                  }
               }
            end
         }
      end
      # -- replace original @tests with tmp_tests array
      @tests = tmp_tests.uniq
   end
end

# -- load test: populate a hash with right entries and create a Test object for it
def load_test(tc)
   data = Hash.new
   File.open(tc, "r") do |infile|
      while (line = infile.gets)
         test                  = /^.*\/(.*\.rb)/.match(tc)[1]
         data['execute_class'] = /^(.*\.rb)/.match(tc)[1]
         data['path']          = /(.*)\/#{test}/.match(tc)[1]
         data['execute_args']  = /^#[\s]*@executeArgs[\s]*(.*)/.match(line)[1] if /^#[\s]*@executeArgs/.match(line)
         data['author']        = /^#[\s]*@author[\s]*(.*)/.match(line)[1] if /^#[\s]*@author/.match(line)
         data['keywords']      = /^#[\s]*@keywords[\s]*(.*)/.match(line)[1].gsub(/,/,'').split if /^#[\s]*@keywords/.match(line)
         data['description']   = /^#[\s]*@description[\s]*(.*)/.match(line)[1] if /^#[\s]*@description/.match(line)
      end
   end
   @tests << Test.new(data, @test_data) if data['keywords'] != nil and data['keywords'] != ""
end

# -- find tests and load them one by one, applying keyword-filter at the end
desc "-- find all tests..."
task :find_all do
   FileList["#{@test_dir}/**/*test.rb"].exclude(@excludes).each { |tc_name|
      load_test(tc_name)
   }
   filter_by_keywords
end

# -- print tests
desc "-- print tests..."
task :print_human do
   Rake::Task["find_all"].invoke
   each { |t|
      begin
         puts t.to_s
      rescue => e
         puts "-- ERROR: " + e.inspect
         puts "   (in test: #{t.execute_class})"
      end
   }
end

# -- run all tests
desc "-- run all tests..."
task :run do
   # -- first, let's setup/cleanup reports_dir
   prepare_reports_dir
   # -- now, find all tests
   Rake::Task["find_all"].invoke
   tStart = Time.now
   # -- let's run each test now
   each { |t|
      begin
         t.validate
         # -- do we run test more than once if it failed first time ?
         if (t.exit_status == @test_data['test_exit_status_failed']) and (@test_data['test_retry'])
            puts("-- first attempt failed, will try again...")
            t.validate
            @tests_retried_counter += 1
         end
      rescue => e
         puts "-- ERROR: " + e.inspect
         puts "   (in test: #{t.execute_class})"
      ensure
         @executed_tests += 1
      end
   }
   tFinish = Time.now
   @execution_time = tFinish - tStart
   clean_exit
end

# -- total by exit status
def all_by_exit_status(status)
   a = Array.new
   each { |t|
      a << t if t.exit_status == status
   }
   return a
end

# -- what do we do on exit ?
def clean_exit
   passed  = all_by_exit_status(@test_data['test_exit_status_passed'])
   failed  = all_by_exit_status(@test_data['test_exit_status_failed'])
   skipped = all_by_exit_status(@test_data['test_exit_status_skipped'])
   @test_data.merge!({'execution_time' => @execution_time, 'passed' => passed, 'failed' => failed, 'skipped' => skipped})
   Publisher.new(@test_data).publish_reports
   puts("\n==> DONE\n\n")
   puts("      -- execution time  : #{@execution_time.to_s} secs\n")
   puts("      -- tests executed  : #{@executed_tests.to_s}\n")
   puts("      -- reports prepared: #{@test_data['reports_dir']}\n")
   puts("      -- tests passed    : #{passed.length.to_s}\n")
   puts("      -- tests failed    : #{failed.length.to_s}\n")
   puts("      -- tests skipped   : #{skipped.length.to_s}\n")
   if @test_data['test_retry']
      puts("      -- tests re-tried  : #{@tests_retried_counter.to_s}\n")
   end
   if failed.length > 0
      puts("\n\n==> STATUS: [ some tests failed - execution failed ]\n")
      exit(1)
   end
   exit(0)
end

#
# ::: Publisher [ creating report files ] :::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
#
class Publisher
   def initialize(test_data)
      @test_data = test_data
      @total     = @test_data['passed'].length + @test_data['failed'].length + @test_data['skipped'].length
   end

   def write_file(file, data)
      File.open(file, 'w') {|f| f.write(data) }
   end

   def create_html_reports(status)
      output  = "<html><body>\n\nTests that #{status}:<br><br><table><tr><td>test</td><td>time</td></tr><tr></tr>\n"
      @test_data[status].each { |t|
         output += "<tr><td><a href='#{t.execute_class}.html'>#{t.execute_class}</a></td><td>#{t.execution_time}</td></tr>\n"
      }
      output += "</table></body></html>"
      write_file(@test_data['reports_dir'] + "/#{status}.html", output)
   end

   def publish_reports
      # -- remove reports dir if it exists, then create it
      # -- create an xml file
      document  = "<?xml version=\"1.0\" encoding=\"UTF-8\" ?>\n"
      document += "<testsuites>\n"
      document += "   <testsuite successes='#{@test_data['passed'].length}'"
      document += "   skipped='#{@test_data['skipped'].length}' failures='#{@test_data['failed'].length}'"
      document += "   time='#{@test_data['execution_time']}' name='FunctionalTestSuite' tests='#{@total}'>\n"
      if @test_data['passed'].length > 0
         @test_data['passed'].each  { |t|
            document += "   <testcase name='#{t.execute_class}' classname='#{@test_data['xml_report_class_name']}' time='#{t.execution_time}'>\n"
            document += "      <passed message='Test Passed'><![CDATA[\n\n#{t.output}\n\n]]>\n       </passed>\n"
            document += "   </testcase>\n"
         }
      end
      if @test_data['failed'].length > 0
         @test_data['failed'].each  { |t|
            document += "   <testcase name='#{t.execute_class}' classname='#{@test_data['xml_report_class_name']}' time='#{t.execution_time}'>\n"
            document += "      <error message='Test Failed'><![CDATA[\n\n#{t.output}\n\n]]>\n       </error>\n"
            document += "   </testcase>\n"
         }
      end
      if @test_data['skipped'].length > 0
         @test_data['skipped'].each  { |t|
            document += "   <testcase name='#{t.execute_class}' classname='#{@test_data['xml_report_class_name']}' time='#{t.execution_time}'>\n"
            document += "      <skipped message='Test Skipped'><![CDATA[\n\n#{t.output}\n\n]]>\n       </skipped>\n"
            document += "   </testcase>\n"
         }
      end
      document += "   </testsuite>\n"
      document += "</testsuites>\n"
      # -- write XML report
      write_file(@test_data['reports_dir'] + "/" + @test_data['xml_report_file_name'], document)
      # -- write HTML report
      totals  = "<html><body>\n\nTotal tests: #{@total.to_s}<br>\n"
      totals += "Passed: <a href='passed.html'>#{@test_data['passed'].length.to_s}</a><br>\n"
      totals += "Failed: <a href='failed.html'>#{@test_data['failed'].length.to_s}</a><br>\n"
      totals += "Skipped: <a href='skipped.html'>#{@test_data['skipped'].length.to_s}</a><br>\n"
      totals += "Execution time: #{@test_data['execution_time']}<br>\n</body></html>"
      write_file(@test_data['reports_dir'] + "/report.html", totals)
      # -- create individual html report files complete with test output
      create_html_reports("passed")
      create_html_reports("failed")
      create_html_reports("skipped")
   end
end

#
# ::: Test class [ running a test ] :::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
#
class Test
    attr_accessor :path, :execute_class, :execute_args, :keywords, :description, :author,
                  :exit_status, :output, :execution_time, :test_data
    def initialize(hash, test_data)
       @exit_status    = @output = @path = @execute_class = @execute_args = @keywords = @description = @author = ""
       @test_data      = test_data
       @execution_time = 0.0
       @timeout        = 1200 # miliseconds
       @path           = hash['path']
       @execute_class  = hash['execute_class']
       @execute_args   = hash['execute_args']
       @keywords       = hash['keywords']
       @description    = hash['description']
       @author         = hash['author']
       @cmd            = @execute_class
    end

    # -- we should do something useful here
    def is_valid
       return true
    end

    def validate
       if is_valid
          # -- run the test if its valid
          run
          # -- write entire test output into its own log file
          write_log
       else
          # -- skipping this test
          @exit_status = @test_data['test_exit_status_skipped']
       end
    end

    def write_file(file, data)
       File.open(file, 'w') {|f| f.write(data) }
    end

    def write_log
       d = /^(.*\/).*/.match(@execute_class)[1]
       FileUtils.mkdir_p(@test_data['reports_dir'] + "/#{d}")
       write_file(@test_data['reports_dir'] + "/#{@execute_class}.html", "<html><body><pre>" + @output + "</pre></body></html>")
    end

    def run
       @cmd = @cmd + " " + @execute_args unless @execute_args == ""
       ENV["REPORT_FILE"] = File.join(@test_data['reports_dir'], @execute_class)
       tStart = Time.now
       print("-- #{tStart.strftime('[%H:%M:%S]')} running: [#{@cmd}] ")
       begin
          status = Timeout::timeout(@timeout.to_i) {
             @output      = `ruby #{@cmd} 2>&1`
             @exit_status = case @output
                when /PASSED/ then @test_data['test_exit_status_passed']
                when /FAILED/ then @test_data['test_exit_status_failed']
                else @test_data['test_exit_status_failed']
             end
          }
       rescue Timeout::Error => e
          @output << "\n\n[ TERMINATED WITH TIMEOUT (#{@timeout.to_s}) ]"
          @exit_status = @test_data['test_exit_status_failed']
       ensure
          puts @exit_status
          puts @output if @test_data['output_on']
       end
       tFinish = Time.now
       @execution_time = tFinish - tStart
    end

    def to_s
        s = "\n  path:          " + @path              + "\n"
        if @author != nil
           s += "  author         " + @author              + "\n"
        end
        s += "  execute_class  " + @execute_class       + "\n"
        s += "  execute_args   " + @execute_args        + "\n"
        s += "  keywords       " + @keywords.join(',')  + "\n"
        s += "  description    " + @description         + "\n"
        s += "  exit_status    " + @exit_status.to_s    + "\n"
        s += "  output         " + @output              + "\n"
        s += "  execution_time " + @execution_time.to_s + "\n\n"
        return s
    end
end

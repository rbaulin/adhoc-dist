
# add template dir and put global envs in rakefile (like full host path)
# add local envs (> gitignore) to app folders like credentials, project build path etc

# usage
# rake new app
# cd app; rake build
# cd app; rake add deviceid
# cd app; rake resign
# rake push
# rake reinit

require 'cgi'
require 'dotenv'
env_path = File.join(Rake.original_dir, '.env')
Dotenv.load(env_path)

# https://gist.github.com/seanlinsley/3e49c09dcdfc37f05eb4
# http://stackoverflow.com/questions/1489183/colorized-ruby-output
class String
  def red;            "\e[31m#{self}\e[0m" end
  def green;          "\e[32m#{self}\e[0m" end
  def yellow;          "\e[33m#{self}\e[0m" end
  def blue;           "\e[34m#{self}\e[0m" end
  def magenta;        "\e[35m#{self}\e[0m" end
  def cyan;           "\e[36m#{self}\e[0m" end
  def gray;           "\e[37m#{self}\e[0m" end
  def blink;          "\e[5m#{self}\e[25m" end
  def reverse_color;  "\e[7m#{self}\e[27m" end
end

class String
  def gsub_captured!(regex, replace)
    self.scan(regex).flatten.each { |e|
      # puts "replace #{e} -> #{replace}"
      self.gsub!(e, replace)
    }
    # binding.pry
  end
end

def system_hl(cmd)
  puts cmd.yellow
  system(cmd)
end

MOUNT_URL = "https://rbaulin.github.io/adhoc-dist/"
IPA_NAME = "app.ipa"
PROVISION_NAME = "adhoc.mobileprovision"
INDEX_NAME = "index.html"
MANIFEST_NAME = "manifest.plist"
ARTWORK_NAME = "artwork.png"
UDID_REGEXP = /^\h{40}$/
NAME_REGEXP = /^[a-zA-Z0-9_ ]*$/

def user
  user = ENV["ITC_USER"]
end

def bundle
  user = ENV["ITC_BUNDLE"]
end

def workspace
  workspace = ENV["XC_WORKSPACE"]
end

def project
  project = ENV["XC_PROJECT"]
end

def scheme
  scheme = ENV["XC_SCHEME"]
end

def udid_email
  ENV["UDID_EMAIL"]
end

def preferred_ip_country
  ENV["IP_COUNTRY"]
end

# array of available signing identities
# https://github.com/fastlane/sigh/blob/master/lib/sigh/resign.rb
def installed_identies
  available = `security find-identity -v -p codesigning`
  ids = []
  available.split("\n").each do |current|
    begin
      (ids << current.match(/.*\"(.*)\"/)[1])
    rescue
      nil
    end # the last line does not match
  end

  ids
end

# fetch signing identity from fetched mobile provision
def signing_identity
  provision_path = File.join(Rake.original_dir, PROVISION_NAME)
  provision = File.read(provision_path)
  provision = provision.encode('UTF-8', :invalid => :replace, :replace => nil)
  team_id = provision.scan(/<key>com.apple.developer.team-identifier<\/key>\s+<string>(\w+)<\/string>/).first.first
  matched = installed_identies.find {|i| i =~ /iPhone Distribution.*#{team_id}/ }
end

# instead of using erb with templates write changes directly to index and manifest
def update_info
  require 'zip'
  zip_path = File.join(Rake.original_dir, IPA_NAME)
  zip = Zip::File.open(zip_path)
  info_plist_entry = zip.entries.find {|e| e.name.end_with? ".app/Info.plist" }
  info_plist = zip.read(info_plist_entry)
  # xml or binary plist
  # xml should start with <?xml version="1.0" encoding="UTF-8"?>

  # plutil input stream http://stackoverflow.com/questions/28097737/pipe-data-to-plutil
  # ruby pipe system http://stackoverflow.com/questions/10732626/how-can-i-pass-a-variable-to-a-system-call-in-ruby
  IO.popen('plutil -convert xml1 -r -o - -- -', 'r+') {|f| # don't forget 'r+'
    f.write(info_plist) # you can also use #write
    f.close_write
    info_plist = f.read # get the data from the pipe
  }


  app_name = info_plist.scan(/<key>CFBundleName<\/key>\s+<string>(.+)<\/string>/).flatten.first
  display_name = info_plist.scan(/<key>CFBundleDisplayName<\/key>\s+<string>(.+)<\/string>/).flatten.first
  app_name = display_name if display_name
  build = info_plist.scan(/<key>CFBundleVersion<\/key>\s+<string>([0-9]+)<\/string>/).flatten.first
  version = info_plist.scan(/<key>CFBundleShortVersionString<\/key>\s+<string>([a-zA-Z0-9.]+)<\/string>/).flatten.first
  app_name_version = "#{app_name} #{version} (#{build})"

  relative_path = File.basename(Rake.original_dir) # simplified, without nesting

  manifest_url = File.join(MOUNT_URL, relative_path, MANIFEST_NAME)
  manifest_itms = CGI.escapeHTML("itms-services://?action=download-manifest&url=#{manifest_url}")
  get_udid_url = CGI.escapeHTML("http://get.udid.io/?mail=#{udid_email}&uid=#{app_name}")

  ipa_url = File.join(MOUNT_URL, relative_path, IPA_NAME)
  artwork_url = File.join(MOUNT_URL, relative_path, ARTWORK_NAME)

  index_path = File.join(Rake.original_dir, INDEX_NAME)
  index = File.read(index_path)

  index.gsub!(%r{<title>.*</title>}, %{<title>#{app_name_version}</title>})
  index.gsub!(%r{<footer>.*</footer>}, %{<footer>RAKE ADHOC</footer>})
  index.gsub!(%r{id="title">.*</h1>}, %{id="title">#{app_name_version}</h1>})
  index.gsub!(%r{href=".*"\s+id="install"}, %{href="#{manifest_itms}" id="install"})
  index.gsub!(%r{href=".*"\s+id="udid"}, %{href="#{get_udid_url}" id="udid"})
  File.write(index_path, index)
  puts "updated #{index_path}".yellow

  manifest_path = File.join(Rake.original_dir, MANIFEST_NAME)
  manifest = File.read(manifest_path)
  manifest.gsub_captured!(%r{<string>software-package</string>\s+<key>url</key>\s+<string>(.*)</string>}, ipa_url)
  manifest.gsub_captured!(%r{<string>full-size-image</string>\s+<key>url</key>\s+<string>(.*)</string>}, artwork_url)
  manifest.gsub_captured!(%r{<key>bundle-identifier</key>\s+<string>(.*)</string>}, bundle)
  manifest.gsub_captured!(%r{<key>bundle-version</key>\s+<string>(.*)</string>}, version)
  manifest.gsub_captured!(%r{<key>title</key>\s+<string>(.*)</string>}, app_name)

  File.write(manifest_path, manifest)

  puts "updated #{manifest_path}".yellow
end

# https://github.com/fastlane/fastlane/blob/master/lib/fastlane/actions/register_devices.rb
# https://github.com/fastlane/spaceship/blob/master/lib/spaceship/portal/device.rb
def add_device(udid, name)
  require 'spaceship'
  raise "Invalid device UDID #{udid}".red unless UDID_REGEXP =~ udid
  raise "Invalid device name #{name}".red unless NAME_REGEXP =~ name
  Spaceship.login(user)
  Spaceship.select_team
  Spaceship::Device.create!(name: name, udid: udid)
end

def find_device(udid)
  require 'spaceship'
  raise "Invalid device UDID #{udid}".red unless UDID_REGEXP =~ udid
  Spaceship.login(user)
  Spaceship.select_team
  Spaceship::Device.find(udid)
  device = Spaceship.device.all.find {|d| d.udid == udid }
  device
end

def fetch_provision
  sigh_cmd = "fastlane sigh --adhoc --force"
  sigh_cmd << " --username #{user}"
  sigh_cmd << " --app_identifier #{bundle}"
  sigh_cmd << " --filename #{PROVISION_NAME}"
  sigh_cmd << " --output_path #{Rake.original_dir}"
  system_hl(sigh_cmd)
end

def resign_ipa
  ipa_path = File.join(Rake.original_dir, IPA_NAME)
  unless File.exists? ipa_path
    puts "rake build".yellow
    exit
  end

  provision_path = File.join(Rake.original_dir, PROVISION_NAME)
  unless File.exists? provision_path
    puts "rake resign fetch".yellow
    exit
  end

  resign_cmd = "fastlane sigh resign #{ipa_path}"
  resign_cmd << " --signing_identity '#{signing_identity}'"
  resign_cmd << " --provisioning_profile #{provision_path}"
  system_hl(resign_cmd)
end

def validate_ip
  return unless preferred_ip_country

  require 'json'
  ip_info_raw = %x{curl http://ip-api.com/json --silent}
  ip_info = JSON.parse(ip_info_raw)
  ip_country = ip_info["countryCode"]

  unless ip_country.downcase == preferred_ip_country.downcase
    raise "ip country #{ip_country} differ from preferred #{preferred_ip_country}"
  end
end


task :bootstrap do
  # build ipa
  # fetch profile
  # resign
  # bump
end

task :build do
  gym_cmd = "fastlane gym"
  gym_cmd << " --workspace #{workspace}" if workspace
  gym_cmd << " --project #{project}" if project
  gym_cmd << " --scheme '#{scheme}'" if scheme
  gym_cmd << " --configuration 'Ad Hoc'"
  gym_cmd << " --output_name #{IPA_NAME}"
  gym_cmd << " --output_directory #{Rake.original_dir}"
  gym_cmd << " --use_legacy_build_api"
  system_hl(gym_cmd)

  resign_ipa
  update_info
end

task :bump do
  update_info
end

task :find do
  validate_ip
  device = find_device(ARGV[1])
  if device
    puts "found device record #{device}".green
  else
    puts "device not found".red
  end
  exit # need this to workaround unknown task errors
end

task :add do
  validate_ip
  udid, name = ARGV[1], ARGV[2]
  add_device(udid, name)
  puts "added #{name} (#{udid})".green
  exit # need this to workaround unknown task errors
end

task :resign do
  validate_ip
  if (ARGV[1] == "fetch")
    fetch_provision
  end
  resign_ipa
  update_info
  exit
end

task :push do
  %x{git add . ; git commit -am \"update\"; git push}
end

task :browsersync do
  cmd = "cd #{Rake.original_dir}; browser-sync start --files '**/*.html, **/*.css' --server"
  system_hl(cmd)
end

task :pry do
  require 'pry'
  binding.pry
end

task :default => :push

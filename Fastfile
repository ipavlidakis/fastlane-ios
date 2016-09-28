default_platform :ios

before_all do
  if (ENV['VERIFY_CLEAN_REPO'] == '1' || ENV['VERIFY_CLEAN_REPO'].downcase == 'true')
    ensure_git_status_clean
  end
  
  setup_jenkins
  
  begin
    cocoapods
    increase_build_number
  rescue => ex
    puts("#{ex}")
  end
end

before_each do |lane, options|
  puts "=> Lane working directory: " + Dir.pwd
end

lane :increase_build_number do
  increment_build_number
end

lane :prepare do |options|
  scheme            = options[:scheme]
  name              = options[:name]
  itcScheme         = if options[:itcScheme]; options[:itcScheme] else scheme end
  testFlightUpload  = if options[:testflight]; options[:testflight] else false end
  fabricUpload      = if options[:fabric]; options[:fabric] else false end 
  configuration     = if options[:configuration]; options[:configuration] else scheme end

  certificates(scheme: scheme, itcScheme: itcScheme)
  
  if (ENV['CUSTOM_DEVELOPER_DIR'])
    ENV['DEVELOPER_DIR'] = ENV['CUSTOM_DEVELOPER_DIR']
  end
  
  build(scheme: scheme, name: name, configuration: configuration)
  
  if testFlightUpload
    itc(scheme: scheme, name: name)
  end

  if fabricUpload
    fabric(scheme: scheme, name: name)
  end

  clean_and_finish

end

# [TO BE OVERRIDEN - START]

lane :testing do
  prepare(scheme: 'Testing', name: ENV['PROJECT_NAME'], testflight: false, fabric:true, configuration: 'Testing')
end

lane :staging do
  prepare(scheme: 'Staging', name: ENV['PROJECT_NAME'], testflight: true, fabric:true, configuration: 'Staging')
end

lane :release do
  prepare(scheme: 'Distribution', name: ENV['PROJECT_NAME'], testflight: true, fabric:true, configuration: 'Distribution')
end

# [TO BE OVERRIDEN - END]

lane :update_property do |options|
  key = options[:key] #PRODUCT_BUNDLE_IDENTIFIER
  value = options[:value]

  puts(sh("pwd"))
  project_file = "../#{ENV['PROJECT_NAME']}.xcodeproj/project.pbxproj"
  oldValue = sh("awk -F '=' '/#{key}/ {print $2; exit}' #{project_file}")
  oldValue = oldValue.strip!.tr(';','')
  command = "sed -i '' 's/"
  command << oldValue
  command << "/"
  command << "\"#{value}\";"
  command << '\n'
  command << "/g' #{project_file}"

  puts("Will execute: #{command}")
  sh(command)
end

desc "Fetches the provisioning profiles so you can build locally and deploy to your device"
lane :certificates do |options|
  scheme                        = options[:scheme]
  itcScheme                     = if options[:itcScheme]; options[:itcScheme] else scheme end
  skip_certificate_verification = if options[:skip_certificate_verification]; options[:skip_certificate_verification] else true end
  team_id                       = CredentialsManager::AppfileConfig.try_fetch_value(:team_id)

  puts("Working scheme: #{scheme} and ITCScheme: #{itcScheme} and team_id: #{team_id}")   
  import_certificates(scheme: itcScheme)
  sigh(skip_certificate_verification: skip_certificate_verification, team_id: team_id)
end

desc "Installs bundle certificates"
lane :import_certificates do |options|
  scheme    = options[:scheme]
  name      = "#{ENV["KEYCHAIN_NAME"]}.keychain"
  password  = ENV["KEYCHAIN_PASSWORD"]
  path      = "#{ENV['CERTIFICATES_PATH']}/#{ENV["APP_IDENTIFIER"]}/#{scheme}"
  keypath   = "#{path}/key.p12"
  certpath  = "#{path}/certificate.cer"
  
  puts("Name #{name} and password #{password}")
  puts("Listing available keychains")
  sh("security list-keychains")
  
  begin
    puts("1st try: Looking for #{name} keychain file")
    unlock_keychain(path: "#{name}", password: "#{password}")
  rescue => ex
    begin
      name = "#{name}-db"
      puts("2nd try: Looking for #{name} keychain file")
      unlock_keychain(path: "#{name}", password: "#{password}")
    rescue => exception
      puts("Keychain doesn't exitst. Let's create it. #{exception}")
      name = "#{ENV["KEYCHAIN_NAME"]}.keychain"
      create_keychain(name: "#{name}",default_keychain: true,unlock: true, timeout: 3600,lock_when_sleeps: true, password: "#{password}",)
    end
  end

  begin
    puts("Signing certificate file path: #{certpath}")
    import_certificate(certificate_path: "#{certpath}",keychain_name: "#{name}")
  rescue => ex
    puts("Exception in lane certificates:certificate #{ex}")
  end
 
  begin
    puts("Private key file path: #{keypath}")
    import_certificate(certificate_path: "#{keypath}",certificate_password: ENV["KEY_EXPORT_PASSWORD"],keychain_name: "#{name}")
  rescue => ex
    puts("Exception in lane certificates:private_key #{ex}")
  end
end

desc "Builds the project"
lane :build do |options|
  # signingId = options[:signId]
  scheme              = options[:scheme]
  name                = options[:name]
  configuration       = options[:configuration]
  clean               = if ENV['BUILD_CLEAN_PROJECT']; ENV['BUILD_CLEAN_PROJECT'] else true end
  inlude_bitcode      = if ENV['BUILD_INCLUDE_BITCODE']; ENV['BUILD_INCLUDE_BITCODE'] else true end
  workspace           = if ENV['BUILD_WORKSPACE']; ENV['BUILD_WORKSPACE'] else "#{name}.xcworkspace" end
  output_dir          = if ENV['BUILD_OUTPUT_DIRECTORY']; ENV['BUILD_OUTPUT_DIRECTORY'] else '' end
  output_name         = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{name}.ipa" end
  use_legacy_build_api= if ENV['BUILD_USE_LEGACY_API']; ENV['BUILD_USE_LEGACY_API'] else false end

  gym(
    scheme: scheme,
    configuration: configuration,
    clean: clean,
    include_bitcode: inlude_bitcode,
    workspace: workspace,
    output_directory: output_dir,
    output_name: output_name,
    xcargs: "ARCHIVE=YES", # Used to tell the Fabric run script to upload dSYM file
    use_legacy_build_api: use_legacy_build_api
  )
end

desc "On success build upload sends a slack message"
lane :post_to_slack do |options|
  scheme      = options[:scheme]
  name        = options[:name]
  version     = get_version_number(xcodeproj: "#{name}.xcodeproj")
  build       = get_build_number(xcodeproj: "#{name}.xcodeproj")
  environment = scheme.upcase
  # api         = File.read("../Fitbay/Constants/environment_constants.h").scan(/\d\.*/).join
  destination = options[:destination]

  slack(
    message: "<!here|here>: New :ios: *#{version}* (#{build}) running '#{environment}' has been submitted to *#{destination}*  :rocket:",
  )
end

lane :clean_and_finish do
  project_name        = if ENV['PROJECT_NAME']; ENV['PROJECT_NAME'] else "" end
  output_dir          = if ENV['BUILD_OUTPUT_DIRECTORY']; ENV['BUILD_OUTPUT_DIRECTORY'] else '' end
  output_name         = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{project_name}.ipa" end

  sh("cd ..")
  output_file_name = "#{output_dir}#{output_name}"
  output_dsym_file_name = "#{output_dir}#{project_name}.app.dSYM.zip"

  begin
    sh("pwd")
    puts("Will try to delete:#{output_file_name}")
    File.delete("#{output_file_name}")
    puts("Will try to delete:#{output_dsym_file_name}")
    File.delete("#{output_dsym_file_name}")

    build = Actions.lane_context[Actions::SharedValues::BUILD_NUMBER]
    puts("Will try to commit build[#{build}] number")
    commit_version_bump(message: "Build #{build}")
  rescue => ex
    puts("Clean & Finish lane errored: #{ex}")
  end
end

lane :fabric do |options|
  scheme                    = options[:scheme]
  crashlytics_path          = if ENV['CRASHLYTICS_FRAMEWORK_PATH']; ENV['CRASHLYTICS_FRAMEWORK_PATH'] else '' end
  crashlytics_api_token     = if ENV['CRASHLYTICS_API_TOKEN']; ENV['CRASHLYTICS_API_TOKEN'] else '' end
  crashlytics_build_secret  = if ENV['CRASHLYTICS_BUILD_SECRET']; ENV['CRASHLYTICS_BUILD_SECRET'] else '' end
  project_name              = if ENV['PROJECT_NAME']; ENV['PROJECT_NAME'] else "" end
  output_dir                = if ENV['BUILD_OUTPUT_DIRECTORY']; ENV['BUILD_OUTPUT_DIRECTORY'] else '' end
  output_name               = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{project_name}.ipa" end
  output_file_name          = "#{output_dir}#{output_name}"

  crashlytics(
    crashlytics_path: crashlytics_path,
    api_token: crashlytics_api_token,
    build_secret: crashlytics_build_secret,
    ipa_path: output_file_name,
    notes: "Built with scheme: #{scheme}. Upload file name: #{output_name}",
  )

  post_to_slack(scheme: scheme, destination: "Crashlytics", name: project_name)
end

lane :itc do |options|
  scheme                            = options[:scheme]
  skip_waiting_for_build_processing = if ENV['ITC_SKIP_WAITING']; ENV['ITC_SKIP_WAITING'] else true end 
  project_name                      = if ENV['PROJECT_NAME']; ENV['PROJECT_NAME'] else "" end
  output_dir                        = if ENV['BUILD_OUTPUT_DIRECTORY']; ENV['BUILD_OUTPUT_DIRECTORY'] else '' end
  output_name                       = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{project_name}.ipa" end
  output_file_name                  = "#{output_dir}#{output_name}"

  pilot(ipa: output_file_name, skip_waiting_for_build_processing: skip_waiting_for_build_processing)
  post_to_slack(scheme: scheme, destination: "TestFlight", name: project_name)
end

after_all do |lane|
  message = "Fastlane finished '#{lane}' successfully"

  begin
    notification(message:message)
    if ENV['ENABLED_NOTIFICATIONS']
      slack(message: message , success: true)
    end
  rescue => ex
    puts("After_all #{lane} errored: #{ex}")
  end
end

error do |lane, exception|
  message = "Fastlane #{lane} errored: #{exception}"

  begin
    notification(message:message)
    if ENV['ENABLED_NOTIFICATIONS']
      slack(message: message , success: false)
    end
  rescue => ex
    puts("Error #{lane} errored: #{ex}")
  end
end
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
  increment_build_number({
    build_number: latest_testflight_build_number + 1
  })
end

lane :use_distribution_provisioning_profile do
  update_property(key:"CODE_SIGN_IDENTITY", value: "iPhone Distribution")
end

lane :use_development_provisioning_profile do
  update_property(key:"CODE_SIGN_IDENTITY", value: "iPhone Development")
end

lane :use_sigh_provisioning_profile do
  update_property(key:"CODE_SIGN_IDENTITY", value: ENV["SIGH_CERTIFICATE_ID"])
end

lane :prepare do |options|
  scheme            = options[:scheme]
  name              = options[:name]
  itcScheme         = if options[:itcScheme]; options[:itcScheme] else scheme end
  testFlightUpload  = if options[:testflight]; options[:testflight] else false end
  fabricUpload      = if options[:fabric]; options[:fabric] else false end 
  configuration     = if options[:configuration]; options[:configuration] else scheme end

  update_bundle_id
  update_team
  use_distribution_provisioning_profile

  certificates(scheme: scheme, itcScheme: itcScheme)
  
  if (ENV['CUSTOM_DEVELOPER_DIR'])
    ENV['DEVELOPER_DIR'] = ENV['CUSTOM_DEVELOPER_DIR']
  end
  puts("XCode Path: #{ENV['DEVELOPER_DIR']}")
  
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
  oldValue = ''
  
  begin
    project_file =  Dir["*.xcodeproj"].first || "../#{ENV['PROJECT_NAME']}.xcodeproj"
    project_file = "#{project_file}/project.pbxproj"
    oldValue = sh("awk -F '=' '/#{key}/ {print $2; exit}' #{project_file}")
    oldValue = oldValue.strip!.tr(';','')
  rescue Exception
    oldValue = ''
  end
  
  command = "sed -i '' 's/"
  command << oldValue
  command << "/"
  command << "\"#{value}\";"
  command << '\n'
  command << "/g' #{project_file}"

  puts("Will execute: #{command}")
  sh(command)
end

lane :update_team do |options|
  project_file = Dir["*.xcodeproj"].first || "#{ENV['PROJECT_NAME']}.xcodeproj"
  team_id = CredentialsManager::AppfileConfig.try_fetch_value(:team_id)

  puts("Will update team_id: #{team_id} and path: #{project_file}")

  update_project_team(
    path: project_file,
    teamid: team_id
  )
end

lane :update_bundle_id do |options|
  project_file = Dir["*.xcodeproj"].first || "../#{ENV['PROJECT_NAME']}.xcodeproj"
  plist_file = "#{ENV['PROJECT_NAME']}/Info.plist"
  bundle_id = ENV["APP_IDENTIFIER"]

  puts("Will update bundle_id: #{bundle_id}")

  update_app_identifier(
    plist_path: plist_file, # Path to info plist file, relative to xcodeproj
    app_identifier: bundle_id
  )
end

desc "Fetches the provisioning profiles so you can build locally and deploy to your device"
lane :certificates do |options|
  scheme                        = options[:scheme]
  itcScheme                     = if options[:itcScheme]; options[:itcScheme] else scheme end
  skip_certificate_verification = if options[:skip_certificate_verification]; options[:skip_certificate_verification] else true end
  team_id                       = CredentialsManager::AppfileConfig.try_fetch_value(:team_id)

  puts("Working scheme: #{scheme} and ITCScheme: #{itcScheme} and team_id: #{team_id}")   
  import_certificates(scheme: itcScheme)
  ENV["PROFILE_UUID"] = sigh(skip_certificate_verification: skip_certificate_verification, team_id: team_id)
  puts("Selected UUID: #{ENV["PROFILE_UUID"]}")
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
      create_keychain(name: "#{name}",default_keychain: false, unlock: true, timeout: 3600,lock_when_sleeps: true, password: "#{password}",)
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
  output_dir          = "#{output_dir}#{name}/#{scheme}/"
  output_name         = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{name}.ipa" end
  use_legacy_build_api= if ENV['BUILD_USE_LEGACY_API']; ENV['BUILD_USE_LEGACY_API'] else false end
  toolchain           = if ENV['BUILD_TOOLCHAIN']; ENV['BUILD_TOOLCHAIN'] else false end
  ENV["FINAL_OUTPUT_BUILD_DIRECTORY"] = output_dir
  
  begin
    sh("rm -Rf #{output_dir}")  
  rescue Exception
    puts("Exception occured but nothing to worry about");
  end

  if toolchain
    # use_legacy_build_api = false
    gym(
        scheme: scheme,
        configuration: configuration,
        clean: clean,
        include_bitcode: inlude_bitcode,
        workspace: workspace,
        output_directory: output_dir,
        output_name: output_name,
        xcargs: "ARCHIVE=YES", # Used to tell the Fabric run script to upload dSYM file
        use_legacy_build_api: use_legacy_build_api,
        toolchain: toolchain
      )
  else
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
end

desc "On success build upload sends a slack message"
lane :post_to_slack do |options|
  scheme      = options[:scheme]
  name        = options[:name]
  version     = get_version_number(xcodeproj: "#{name}.xcodeproj")
  build       = get_build_number(xcodeproj: "#{name}.xcodeproj")
  environment = scheme.upcase
  destination = options[:destination]

  slack(
    message: "<!here|here>: New :ios: *#{version}* (#{build}) running '#{environment}' has been submitted to *#{destination}*  :rocket:",
  )
end

lane :clean_and_finish do
  project_name        = if ENV['PROJECT_NAME']; ENV['PROJECT_NAME'] else "" end
  output_dir          = ENV["FINAL_OUTPUT_BUILD_DIRECTORY"]
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
  output_dir                = ENV["FINAL_OUTPUT_BUILD_DIRECTORY"]
  output_name               = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{project_name}.ipa" end
  output_file_name          = if ENV['IPA_OUTPUT_PATH']; ENV['IPA_OUTPUT_PATH'] else "#{output_dir}#{output_name}" end

  puts("Crashlytics IPA path: #{output_file_name}")
  crashlytics(
    crashlytics_path: crashlytics_path,
    api_token: crashlytics_api_token,
    build_secret: crashlytics_build_secret,
    ipa_path: output_file_name,
    notes: "Built with scheme: #{scheme}. Upload file name: #{output_name}",
  )

  if ENV['ENABLED_NOTIFICATIONS']
    post_to_slack(scheme: scheme, destination: "Crashlytics", name: project_name)
  end
end

lane :itc do |options|
  scheme                            = options[:scheme]
  skip_waiting_for_build_processing = if ENV['ITC_SKIP_WAITING']; ENV['ITC_SKIP_WAITING'] else true end 
  project_name                      = if ENV['PROJECT_NAME']; ENV['PROJECT_NAME'] else "" end
  output_dir                        = ENV["FINAL_OUTPUT_BUILD_DIRECTORY"]
  output_name                       = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{project_name}.ipa" end
  output_file_name                  = if ENV['IPA_OUTPUT_PATH']; ENV['IPA_OUTPUT_PATH'] else "#{output_dir}#{output_name}" end

  puts("TestFlight IPA path: #{output_file_name}")
  pilot(ipa: output_file_name, skip_waiting_for_build_processing: skip_waiting_for_build_processing)
  if ENV['ENABLED_NOTIFICATIONS']
    post_to_slack(scheme: scheme, destination: "TestFlight", name: project_name)
  end
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
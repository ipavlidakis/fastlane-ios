default_platform :ios

before_all do
  if (ENV['VERIFY_CLEAN_REPO'] == '1' || ENV['VERIFY_CLEAN_REPO'].downcase == 'true')
    ensure_git_status_clean
  end
  
  setup_jenkins
  
  begin
    ENV["PROJECT_PWD"] = sh("pwd").strip!.sub('fastlane','')
    ENV["PROJECT_PWD"] = Shellwords.escape(ENV["PROJECT_PWD"])
    puts("===> PROJECT_PWD: #{ENV["PROJECT_PWD"]} <===")
  rescue => ex
    puts("#{ex}")
  end
end

lane :increase_build_number do
  increment_build_number({
    build_number: latest_testflight_build_number + 1
  })
end

lane :match_signing do |options|
  configuration = options[:configuration]
  ENV['MATCH_TYPE'] = configuration
  
  team_id = CredentialsManager::AppfileConfig.try_fetch_value(:team_id)
  puts("Will run match now for configuration: #{configuration} and team_id: #{team_id}")
  match(app_identifier: ENV["APP_IDENTIFIER"], team_id: team_id)

  ENV["PROFILE_UUID_NAME"]  = ENV["sigh_#{ENV["APP_IDENTIFIER"]}_#{configuration}_profile-name"]
  ENV["PROFILE_UUID"]       = ENV["sigh_#{ENV["APP_IDENTIFIER"]}_#{configuration}"]
  ENV["MATCH_TEAM_ID"]      = ENV["sigh_#{ENV["APP_IDENTIFIER"]}_#{configuration}_team-id"]

  puts("PROFILE_UUID_NAME: #{ENV["PROFILE_UUID_NAME"]}")
  puts("PROFILE_UUID: #{ENV["PROFILE_UUID"]}")


  puts("MATCH_TEAM_ID: #{ENV["MATCH_TEAM_ID"]}")

  # project_file = "#{ENV["PROJECT_PWD"]}#{ENV['PROJECT_NAME']}.xcodeproj"
  update_team
end

lane :prepare do |options|
  scheme            = options[:scheme]
  name              = options[:name]
  match             = options[:match]
  itcScheme         = if options[:itcScheme]; options[:itcScheme] else scheme end
  testFlightUpload  = if options[:testflight]; options[:testflight] else false end
  fabricUpload      = if options[:fabric]; options[:fabric] else false end 
  configuration     = if options[:configuration]; options[:configuration] else scheme end

  cocoapods

  puts("Project Path : #{ENV["PROJECT_PWD"]}")

  update_bundle_id
  increase_build_number

  match_signing(configuration: match)
  
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
  prepare(scheme: 'Testing', name: ENV['PROJECT_NAME'], testflight: false, fabric:true, configuration: 'Testing', match: 'development')
end

lane :staging do
  prepare(scheme: 'Staging', name: ENV['PROJECT_NAME'], testflight: true, fabric:true, configuration: 'Staging', match: 'appstore')
end

lane :release do
  prepare(scheme: 'Distribution', name: ENV['PROJECT_NAME'], testflight: true, fabric:true, configuration: 'Distribution', match: 'appstore')
end

# [TO BE OVERRIDEN - END]

lane :update_team do |options|
  project_file = "#{ENV['PROJECT_NAME']}.xcodeproj"
  team_id = CredentialsManager::AppfileConfig.try_fetch_value(:team_id)

  puts("Will update team_id: #{team_id} and path: #{project_file}")

  update_project_team(
    path: project_file,
    teamid: team_id
  )
end

lane :update_bundle_id do |options|
  plist_file = if ENV['INFO_PLIST_PATH']; "#{ENV['INFO_PLIST_PATH']}/Info.plist" else "Info.plist" end
  plist_file = "#{ENV["PROJECT_NAME"]}/#{plist_file}"

  bundle_id = ENV["APP_IDENTIFIER"]

  puts("Will update bundle_id: #{bundle_id}")

  update_info_plist(
    plist_path: plist_file, # Path to info plist file, relative to xcodeproj
    app_identifier: bundle_id
  )
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
  output_dir          = "#{output_dir}#{Shellwords.escape(name)}/#{Shellwords.escape(scheme)}/"
  output_name         = if ENV['BUILD_OUTPUT_NAME']; ENV['BUILD_OUTPUT_NAME'] else "#{name}.ipa" end
  use_legacy_build_api= if ENV['BUILD_USE_LEGACY_API']; ENV['BUILD_USE_LEGACY_API'] else false end
  toolchain           = if ENV['BUILD_TOOLCHAIN']; ENV['BUILD_TOOLCHAIN'] else false end
  disable_xcpretty    = if ENV['BUILD_DISABLE_XCPRETTY']; ENV['BUILD_DISABLE_XCPRETTY'] else false end
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
        workspace: "#{workspace}",
        output_directory: "#{output_dir}",
        output_name: "#{output_name}",
        xcargs: "ARCHIVE=YES", # Used to tell the Fabric run script to upload dSYM file
        use_legacy_build_api: use_legacy_build_api,
        toolchain: toolchain,
        disable_xcpretty: disable_xcpretty
      )
  else
    gym(
      scheme: scheme,
      configuration: configuration,
      clean: clean,
      include_bitcode: inlude_bitcode,
      workspace: "#{workspace}",
      output_directory: "#{output_dir}",
      output_name: "#{output_name}",
      xcargs: "ARCHIVE=YES", # Used to tell the Fabric run script to upload dSYM file
      use_legacy_build_api: use_legacy_build_api,
      disable_xcpretty: disable_xcpretty
    )
  end

  copy_artifacts_to_jenkins_workspace
end

desc "On success build upload sends a slack message"
lane :post_to_slack do |options|
  begin
    scheme      = options[:scheme]
    name        = options[:name]
    version     = get_version_number
    build       = get_build_number
    environment = scheme.upcase
    destination = options[:destination]

    slack(
      message: "<!here|here>: New :ios: *#{name}* *#{version}* (#{build}) running '#{environment}' has been submitted to *#{destination}*  :rocket:",
    )
  rescue => ex
    slack(
      message: "<!here|here>: New :ios: *#{name}* running '#{environment}' has been submitted to *#{destination}*  :rocket:",
    )
  end
end

lane :copy_artifacts_to_jenkins_workspace do
  source_dir = ENV["FINAL_OUTPUT_BUILD_DIRECTORY"]
  destination_dir = ENV['WORKSPACE']

  if destination_dir
    begin
      sh("cp #{source_dir}*.ipa \"#{destination_dir}\"/")
      sh("cp #{source_dir}*.dSYM.zip \"#{destination_dir}\"/")
    rescue => ex
      puts("copy_artifacts_to_jenkins_workspace lane errored: #{ex}")
    end
  end
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

  apple_id = CredentialsManager::AppfileConfig.try_fetch_value(:apple_id)
  team_id = CredentialsManager::AppfileConfig.try_fetch_value(:itc_team_id)
  puts("TestFlight IPA path: #{output_file_name}")
  pilot(
    ipa: output_file_name, 
    skip_waiting_for_build_processing: skip_waiting_for_build_processing,
    username: apple_id,
    team_id: team_id
    )
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
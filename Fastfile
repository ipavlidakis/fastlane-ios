default_platform :ios

# ==== Header Lanes ==== #

before_all do
  # ENV["DELIVER_ITMSTRANSPORTER_ADDITIONAL_UPLOAD_PARAMETERS"] = "-t DAV"
  if (ENV['VERIFY_CLEAN_REPO'] == '1' || ENV['VERIFY_CLEAN_REPO'].downcase == 'true')
    ensure_git_status_clean
  end
  setup_jenkins
  cocoapods
  increment_build_number
end

before_each do |lane, options|
  puts "=> Lane working directory: " + Dir.pwd
end

# ==== Building Lanes ==== #

lane :prepare do |options|
  scheme           = options[:scheme]
  name             = options[:name]
  testFlightUpload = options[:testflight]
  fabricUpload     = options[:fabric]

  certs(scheme: scheme)
  build(scheme: scheme, name: name)
  if testFlightUpload
    itc(scheme: scheme, name: name)
  end

  if fabricUpload
    fabric(scheme: scheme, name: name)
  end
  clean_and_finish
end

lane :testing do
  prepare(scheme: 'Testing', name: ENV['PROJECT_NAME'], testflight: false, fabric:true)
end

lane :staging do
  scheme = 'Staging'
  prepare(scheme: 'Staging', name: ENV['PROJECT_NAME'], testflight: true, fabric:true)
end

lane :release do
  prepare(scheme: 'Release', name: ENV['PROJECT_NAME'], testflight: true, fabric:true)
end

# ==== Utilities Lanes ==== #

desc "Fetches the provisioning profiles so you can build locally and deploy to your device"
lane :certs do |options|
  scheme = options[:scheme]

  if scheme == 'Testing'
    match(type: 'development',readonly: true)
  elsif scheme == 'Staging'
    match(type: 'adhoc',readonly: true)
  else
    match(type: 'appstore',readonly: true)
  end
end

desc "Builds the project"
desc "- Scheme : the scheme to use"
desc "- signingID : code signing identity"
desc "- name : Project's and XCWorkspace filename without extensions"
lane :build do |options|
  scheme = options[:scheme]
  # signingId = options[:signId]
  name = options[:name]

  gym(
    scheme: "#{name} #{scheme}",
    configuration: scheme,
    clean: true,
    include_bitcode: false,
    workspace: "#{name}.xcworkspace",
    output_directory: ENV['OUTPUT_DIRECTORY'],
    output_name: "#{name}.ipa",
    # codesigning_identity: "#{signingId}",
    xcargs: "ARCHIVE=YES", # Used to tell the Fabric run script to upload dSYM file
    use_legacy_build_api: ENV['LEGACY_BUILD_API'] ? ENV['LEGACY_BUILD_API'] : false
  )
end

desc "On success build upload sends a slack message"
desc "- Scheme : the scheme to use"
desc "- name : Project's and xcodeproj filename without extensions"
desc "- destination : Deployment target"
lane :post_to_slack do |options|
  scheme      = options[:scheme]
  name        = options[:name]
  version     = get_version_number(xcodeproj: "#{name}.xcodeproj")
  build       = get_build_number(xcodeproj: "#{name}.xcodeproj")
  environment = scheme.upcase
  # api         = File.read("../Fitbay/Constants/environment_constants.h").scan(/\d\.*/).join
  destination = options[:destination]

  isSlackEnabled = ENV['SLACK_ENABLED']?:true
  if :isSlackEnabled do
    slack(
      message: "<!here|here>: New :ios: *#{version}* (#{build}) running '#{environment}' has been submitted to *#{destination}*  :rocket:",
    )
  end
end

lane :clean_and_finish do
  File.delete('../' + ENV['OUTPUT_DIRECTORY'] + ENV['PROJECT_NAME'] + '.ipa')
  File.delete('../' + ENV['OUTPUT_DIRECTORY'] + ENV['PROJECT_NAME'] + '.app.dSYM.zip')

  build = Actions.lane_context[Actions::SharedValues::BUILD_NUMBER]
  commit_version_bump(
    message: "Build #{build}"
  )
end

# ==== Uploader Lanes ==== #

lane :fabric do |options|
  scheme = options[:scheme]
  name = options[:name]

  crashlytics(
    crashlytics_path: ENV['CRASHLYTICS_FRAMEWORK_PATH'],
    api_token: ENV['CRASHLYTICS_API_TOKEN'],
    build_secret: ENV['CRASHLYTICS_BUILD_SECRET'],
    ipa_path: ENV['OUTPUT_DIRECTORY'] + "#{name}.ipa",
    notes: "Running on #{scheme}",
  )

  post_to_slack(scheme: scheme, destination: "Crashlytics", name: name)
end

lane :itc do |options|
  scheme = options[:scheme]
  name = options[:name]

  pilot(
    ipa: "../#{name}.ipa",
    skip_waiting_for_build_processing: true,
  )

  post_to_slack(scheme: scheme, destination: "TestFlight", name: name)
end

# ==== Footer Lanes ==== #

# This lane is called, only if the executed lane was successful
after_all do |lane|
  message = "Fastlane finished '#{lane}' successfully"
  notification(message:message)

  isSlackEnabled = ENV['SLACK_ENABLED']?:true
  if :isSlackEnabled do
    slack(message: message , success: true)
  end
end

error do |lane, exception|
  message = "Fastlane '#{lane}' errored" + exception.message
  notification(message:message)

  isSlackEnabled = ENV['SLACK_ENABLED']?:true
  if :isSlackEnabled do
    slack(message: message , success: false)
  end
end

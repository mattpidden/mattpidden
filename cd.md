## Mobile App Continuous Deployment to App Stores
Continuous Deployment (CD) for mobile apps streamlines the process of building, testing, and deploying applications directly to app stores like the Apple App Store and Google Play Store. CD is valuable across all mobile development frameworks, including Swift, Flutter, Java, Kotlin, and React Native.
### Benifits of CD for Mobile Apps
- The ability to build an iOS app, without needing to own a Mac
- Quicker, easier, more reliable than building and deploying manually
### General Steps of Mobile App Deployment
1. Compile your application for the appropriate platform (iOS or Android).
2. Deploy the compiled app to the targeted app store either the Apple App Store or Google Play Store.

### Prerequisites
Before you can start creating CD workflows, you will need a developer account for any app stores you want to deploy to (Google Play, Apple App Store). Please get in contact with the Tech Hub in MVB, and they can provide you with the accounts. Do **NOT** pay for the accounts yourself.

If you are setting up the workflow for iOS distribution, you will need to use a Mac when sourcing some of the GitHub Secrets


## Deploy to iOS via Apple App Store CD Workflow
In this example we take a look at building an iOS app, and deploying via Apple App Store. The `yaml` examples have a lot of comments, `echo` and `ls` commands in order to maintain readability, and for easy debugging.

To build and deploy an iOS app you need signing certificates, profiles, keychain passwords, API keys and an `exportOptions.plist` file.
We use GitHub Secrets to store these confidential values securely. For more details on how to get, encode, and upload all of these, see [Setting up GitHub Secrets etc](#setting-up-github-secrets-etc)
1. Start the CD pipeline with a designated trigger, often a pull request merge to the main branch.
```yaml
on:
  pull_request:
    branches:
      - main
```
2. Specify the operating system that the CD pipeline will use, such as macOS for iOS builds
```yaml
runs-on: macos-latest
```
3. Pull the latest code from the designated branch,
```yaml
- name: Checkout
  uses: actions/checkout@v3
```
4. Download and install all necessary packages and dependencies
```yaml
# FOR A FLUTTER APP
- name: Install Flutter
  uses: subosito/flutter-action@v2
  with:
   channel: 'stable'
   cache: true
- name: Install Dependencies  
  run: flutter pub get
```
```yaml
# FOR A REACT NATIVE APP
- name: Set up Node.js
  uses: actions/setup-node@v3
  with:
    node-version: '16'
    cache: 'npm'     
- name: Install Dependencies
  run: npm install
```
```yaml
# FOR A SWIFT APP
- name: Set Xcode Version (if needed)
  run: sudo xcode-select -s /Applications/Xcode_14.0.app
- name: Install Dependencies
  run: pod install
```
5. Load all required app signing certificates, keys, and provisioning profiles to authorize the app build for the selected platform
```yaml
- name: Import Certificates and Provisioning Profiles
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    BUILD_CERTIFICATE_BASE64: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_BASE64 }} 
    P12_PASSWORD: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_PASSWORD }}     
    BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.iOS_PROVISION_PROFILE_BASE64 }}       
    KEYCHAIN_PASSWORD: ${{ secrets.iOS_KEYCHAIN_PASSWORD }}
  run: |   
    echo "Certificate Base64 length: ${#BUILD_CERTIFICATE_BASE64}"
    echo "Prov ision Profile Base64 length: ${#BUILD_PROVISION_PROFILE_BASE64}"

    # Set paths for certificate and profile files
    CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12                
    PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
    KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

    # Decode and save certificates and provisioning profiles from secrets
    echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
    echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

    ls -l $CERTIFICATE_PATH
    ls -l $PP_PATH
    
    file $CERTIFICATE_PATH
    file $PP_PATH

    # Create and unlock a keychain to store the certificate
    security create-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH
    security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
    security unlock-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH

    ls -l $KEYCHAIN_PATH
    security show-keychain-info $KEYCHAIN_PATH

    # Import certificate into the keychain
    security import $CERTIFICATE_PATH -P $P12_PASSWORD -A -t cert -f pkcs12 -k $KEYCHAIN_PATH            
    security list-keychains -d user -s $KEYCHAIN_PATH

    # Move the mobile provisioning profile to the correct folder
    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles
    ls -l ~/Library/MobileDevice/Provisioning\ Profiles
```
6. Retrieve API keys or access tokens required to communicate with the target App Store
```yaml
- name: Decode App Store Connect private key file and save it
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    API_KEY_BASE64: ${{ secrets.iOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64 }}
    API_KEY: ${{ secrets.iOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
    # Create directory for private key storage
    mkdir -p ~/private_keys
    # Decode and save key
    echo -n "$API_KEY_BASE64" | base64 --decode -o ~/private_keys/AuthKey_$API_KEY.p8

```
7. Compile the app for the target platform
```yaml
# FOR A FLUTTER APP
- name: Build iOS application archive
  run: |
    # Compile Flutter app into .ipa file
    flutter build ipa --release --export-options-plist=ios/exportOptions.plist
```
```yaml
# FOR A REACT NATIVE APP
- name: Build iOS application archive
  run: |
    # Navigate to the iOS directory and install dependencies
    cd ios && pod install && cd ..

    # Build and archive the app in one step
    xcodebuild -workspace ios/YourApp.xcworkspace \
               -scheme YourAppScheme \
               -sdk iphoneos \
               -configuration Release \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive archive

    # Export the .ipa
    xcodebuild -exportArchive \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive \
               -exportOptionsPlist ios/exportOptions.plist \
               -exportPath ${{ github.workspace }}/build/ios/ipa
```
```yaml
# FOR A SWIFT APP
- name: Build iOS application archive
  run: |
    # Build and archive the app in one step
    xcodebuild -project YourApp.xcodeproj \
               -scheme YourAppScheme \
               -sdk iphoneos \
               -configuration Release \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive archive

    # Export the .ipa
    xcodebuild -exportArchive \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive \
               -exportOptionsPlist exportOptions.plist \
               -exportPath ${{ github.workspace }}/build/ios/ipa
```
8. Upload the compiled build to the target App Store.
```yaml
- name: Upload to App Store Connect
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    ISSUER_ID: ${{ secrets.IOS_APPSTORE_CONNECT_ISSUER_ID }}          
    API_KEY: ${{ secrets.IOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
      # Upload the .ipa file using the App Store Connect API key and issuer ID
      xcrun altool --upload-app -f build/ios/ipa/*.ipa -t ios --apiKey $API_KEY --apiIssuer "$ISSUER_ID"
```
9. Remove any sensitive data, temporary files, and credentials from the CD environment to maintain security.
```yaml
- name: Clean up keychain and provisioning profile
      if: ${{ always() }}
      run: |
        # Delete the keychain and remove the provisioning profile after upload
        security delete-keychain $RUNNER_TEMP/app-signing.keychain-db
        rm ~/Library/MobileDevice/Provisioning\ Profiles/build_pp.mobileprovision
```

## Deploy to Android via Google Play Store CD Workflow
In this example we take a look at building an Android app, and deploying via Google Play Store. The `yaml` examples have a lot of comments, `echo` and `ls` commands in order to maintain readability, and for easy debugging.
The examples below are for Kotlin and Java projects. If using Flutter or React Native, you will need to use some of the examples from the iOS examples.

To build and deploy an Android app you need signing certificates, profiles, keychain passwords, API keys and an `exportOptions.plist` file.
We use GitHub Secrets to store these confidential values securely. For more details on how to get, encode, and upload all of these, see [Setting up GitHub Secrets etc](#setting-up-github-secrets-etc)
1. Start the CD pipeline with a designated trigger, often a pull request merge to the main branch.
```yaml
on:
  pull_request:
    branches:
      - main
```
2. Specify the operating system that the CD pipeline will use, such as Linux for Android.
```yaml
runs-on: ubuntu-latest
```
3. Pull the latest code from the designated branch,
```yaml
- name: Checkout
  uses: actions/checkout@v3
```
4. Download and install all necessary packages and dependencies
```yaml
# FOR A KOTLIN OR JAVA APP
- name: Set up JDK
  uses: actions/setup-java@v3
  with:
    java-version: '11'        
    cache: 'gradle'              
- name: Install Dependencies
  run: ./gradlew build -x test 
```

5. Load all required app signing certificates, keys, and provisioning profiles to authorize the app build for the selected platform
```yaml
- name: Import Certificates and Provisioning Profiles
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    BUILD_CERTIFICATE_BASE64: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_BASE64 }} 
    P12_PASSWORD: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_PASSWORD }}     
    BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.iOS_PROVISION_PROFILE_BASE64 }}       
    KEYCHAIN_PASSWORD: ${{ secrets.iOS_KEYCHAIN_PASSWORD }}
  run: |   
    echo "Certificate Base64 length: ${#BUILD_CERTIFICATE_BASE64}"
    echo "Prov ision Profile Base64 length: ${#BUILD_PROVISION_PROFILE_BASE64}"

    # Set paths for certificate and profile files
    CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12                
    PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
    KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

    # Decode and save certificates and provisioning profiles from secrets
    echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
    echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

    ls -l $CERTIFICATE_PATH
    ls -l $PP_PATH
    
    file $CERTIFICATE_PATH
    file $PP_PATH

    # Create and unlock a keychain to store the certificate
    security create-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH
    security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
    security unlock-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH

    ls -l $KEYCHAIN_PATH
    security show-keychain-info $KEYCHAIN_PATH

    # Import certificate into the keychain
    security import $CERTIFICATE_PATH -P $P12_PASSWORD -A -t cert -f pkcs12 -k $KEYCHAIN_PATH            
    security list-keychains -d user -s $KEYCHAIN_PATH

    # Move the mobile provisioning profile to the correct folder
    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles
    ls -l ~/Library/MobileDevice/Provisioning\ Profiles
```
6. Retrieve API keys or access tokens required to communicate with the target App Store
```yaml
- name: Decode App Store Connect private key file and save it
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    API_KEY_BASE64: ${{ secrets.iOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64 }}
    API_KEY: ${{ secrets.iOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
    # Create directory for private key storage
    mkdir -p ~/private_keys
    # Decode and save key
    echo -n "$API_KEY_BASE64" | base64 --decode -o ~/private_keys/AuthKey_$API_KEY.p8

```
7. Compile the app for the target platform
```yaml
# FOR A FLUTTER APP
- name: Build iOS application archive
  run: |
    # Compile Flutter app into .ipa file
    flutter build ipa --release --export-options-plist=ios/exportOptions.plist
```
```yaml
# FOR A REACT NATIVE APP
- name: Build iOS application archive
  run: |
    # Navigate to the iOS directory and install dependencies
    cd ios && pod install && cd ..

    # Build and archive the app in one step
    xcodebuild -workspace ios/YourApp.xcworkspace \
               -scheme YourAppScheme \
               -sdk iphoneos \
               -configuration Release \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive archive

    # Export the .ipa
    xcodebuild -exportArchive \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive \
               -exportOptionsPlist ios/exportOptions.plist \
               -exportPath ${{ github.workspace }}/build/ios/ipa
```
```yaml
# FOR A SWIFT APP
- name: Build iOS application archive
  run: |
    # Build and archive the app in one step
    xcodebuild -project YourApp.xcodeproj \
               -scheme YourAppScheme \
               -sdk iphoneos \
               -configuration Release \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive archive

    # Export the .ipa
    xcodebuild -exportArchive \
               -archivePath ${{ github.workspace }}/build/YourApp.xcarchive \
               -exportOptionsPlist exportOptions.plist \
               -exportPath ${{ github.workspace }}/build/ios/ipa
```
8. Upload the compiled build to the target App Store.
```yaml
- name: Upload to App Store Connect
  # This env section fetches values from GitHub secrets and saves them to variables
  env:
    ISSUER_ID: ${{ secrets.IOS_APPSTORE_CONNECT_ISSUER_ID }}          
    API_KEY: ${{ secrets.IOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
      # Upload the .ipa file using the App Store Connect API key and issuer ID
      xcrun altool --upload-app -f build/ios/ipa/*.ipa -t ios --apiKey $API_KEY --apiIssuer "$ISSUER_ID"
```
9. Remove any sensitive data, temporary files, and credentials from the CD environment to maintain security.
```yaml
- name: Clean up keychain and provisioning profile
      if: ${{ always() }}
      run: |
        # Delete the keychain and remove the provisioning profile after upload
        security delete-keychain $RUNNER_TEMP/app-signing.keychain-db
        rm ~/Library/MobileDevice/Provisioning\ Profiles/build_pp.mobileprovision
```
## Setting up GitHub Secrets etc
To build and deploy to App Stores you need certificates, key, files etc. 
### Create exportOptions.plist
### Secrets for iOS Deployment
To build and deploy to iOS via Apple App Store, you need 7 github secrets.
| GitHub Secret | Description | How To |
| :------------ | :--------- | :----- |
| `IOS_APPSTORE_CONNECT_API_KEY_ID` | This is the App Store Connect API Key ID, which is used to authenticate API requests to App Store Connect. It’s needed for uploading your `.ipa` file to App Store Connect. | 1. Log in to App Store Connect. Go to Users and Access > Integrations > App Store Connect API. 2. If you don’t have an API key yet, create one by clicking the `+` button. 3. Copy the Key ID provided – this is your `IOS_APPSTORE_CONNECT_API_KEY_ID`. |
| `IOS_APPSTORE_CONNECT_ISSUER_ID` | The Issuer ID is a unique identifier for your App Store Connect organization, necessary for authenticating API requests along with the API Key ID. | x |
| `IOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64` | App Store Connect API Private Key in Base64 format                                   | x |
| `IOS_KEYCHAIN_PASSWORD` | This is a password for the temporary keychain                                        | x |
| `IOS_P12_DISTRIBUTION_CERT_BASE64` | .p12 Distribution Certificate in Base64 format, used to sign app for distribution    | x |
| `IOS_P12_DISTRIBUTION_CERT_PASSWORD` | The password used to access your .p12 distribution certificate                       | x |
| `IOS_PROVISION_PROFILE_BASE64` | This is your iOS Provisioning Profile in Base64 format                               | x |


**`IOS_APPSTORE_CONNECT_ISSUER_ID**
1. In App Store Connect, go to Users and Access > Integrations > App Store Connect API. 2. The Issuer ID is listed in the API Keys section at the top.

**`IOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64`**
This is your App Store Connect API Private Key in Base64 format. It’s used with the Key ID and Issuer ID to securely authenticate uploads to App Store Connect.

How to Find:
1. When creating an API key in App Store Connect (as described above), download the `.p8` file – this file is your private key.
2. Important: Do not share or expose this key publicly. Store it securely.
3. To encode this key in Base64 (required for GitHub Actions):
   - Run the following command in the terminal:
     ```bash
     base64 -i AuthKey_ABC123XYZ.p8 -o AuthKey_ABC123XYZ.p8.b64
     ```
   - Copy the Base64-encoded contents from `AuthKey_ABC123XYZ.p8.b64` and set it as the value for `IOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64`.

---

**`IOS_KEYCHAIN_PASSWORD`**
This is a password for the temporary keychain created in GitHub Actions to store your code-signing certificates and keys during the build process. This password can be any strong, random string and is used to secure the keychain on the runner.

How to Set:
- Generate a strong password manually or with a password generator.
- Add this password directly as a GitHub secret.

**`IOS_P12_DISTRIBUTION_CERT_BASE64`**
This is your iOS Distribution Certificate in Base64 format, needed to sign the app for App Store distribution.

How to Find:
1. Open Keychain Access on your Mac.
2. Locate your iOS Distribution Certificate (issued by Apple).
3. Export the certificate and private key as a `.p12` file:
   - Right-click the certificate and choose Export.
   - Save it as a `.p12` file with a password (you’ll use this password for `IOS_P12_DISTRIBUTION_CERT_PASSWORD`).
4. Convert the `.p12` file to Base64 for GitHub Actions:
   ```bash
   base64 -i YourCertificate.p12 -o YourCertificate.p12.b64
   ```
5. Copy the contents of `YourCertificate.p12.b64` and set it as `IOS_P12_DISTRIBUTION_CERT_BASE64`.

**`IOS_P12_DISTRIBUTION_CERT_PASSWORD`**
The password used when exporting your .p12 distribution certificate. This allows the certificate to be imported into the GitHub Actions keychain.
How to Set:
1. When you exported your certificate from Keychain Access, you created this password.
2.Store this password directly as a GitHub secret.



**`IOS_PROVISION_PROFILE_BASE64`**
This is your iOS Provisioning Profile in Base64 format, which matches your app’s bundle identifier and distribution certificate for App Store distribution.
How to Find:
1. In Apple Developer Center, go to Profiles under Certificates, Identifiers & Profiles.
2. Download the appropriate Provisioning Profile for App Store distribution (must match the app’s bundle ID).
3. Convert the `.mobileprovision` file to Base64
4. Copy the contents of `YourProvisioningProfile.mobileprovision.b64` and set it as `IOS_PROVISION_PROFILE_BASE64`.

### Secrets for Android Deployment

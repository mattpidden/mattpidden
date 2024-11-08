## Continuous Deployment to App Stores
Continuous Deployment (CD) for mobile apps streamlines the process of building, testing, and deploying applications directly to app stores like the Apple App Store and Google Play Store. CD is valuable across all mobile development frameworks, including Swift, Flutter, Java, Kotlin, and React Native.
### Benifits of CD for Mobile Apps
- The ability to build an iOS app, without needing to own a Mac.
- Quicker, easier, more reliable than building and deploying manually
### General Steps of Mobile App Deployment
1. Compile your application for the appropriate platform (iOS or Android).
2. Deploy the compiled app to the targeted app store either the Apple App Store or Google Play Store.

### Example CD Workflow
In this example we take a look at building a Flutter app, and deploying to iOS via Apple App Store. The overall structure will look very similar for other frameworks, and target platforms.
1. Start the CD pipeline with a designated trigger, often a code push or pull request merge to the main branch.
```yaml
on:
  pull_request:
```
2. Specify the operating system that the CD pipeline will use, such as macOS for iOS builds or Linux for Android.
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
- name: Install Flutter
  uses: subosito/flutter-action@v2
  with:
   channel: 'stable'
   cache: true
- name: Get dependencies  
  run: flutter pub get
```
5. Load all required app signing certificates, keys, and provisioning profiles to authorize the app build for the selected platform
```yaml
- name: Import Certificates and Provisioning Profiles
  env:
    BUILD_CERTIFICATE_BASE64: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_BASE64 }} 
    P12_PASSWORD: ${{ secrets.iOS_P12_DISTRIBUTION_CERT_PASSWORD }}     
    BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.iOS_PROVISION_PROFILE_BASE64 }}       
    KEYCHAIN_PASSWORD: ${{ secrets.iOS_KEYCHAIN_PASSWORD }}
  run: |   
    echo "Certificate Base64 length: ${#BUILD_CERTIFICATE_BASE64}"
    echo "Prov ision Profile Base64 length: ${#BUILD_PROVISION_PROFILE_BASE64}"
    CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12                
    PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
    KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

    echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
    echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

    ls -l $CERTIFICATE_PATH
    ls -l $PP_PATH
    
    file $CERTIFICATE_PATH
    file $PP_PATH

    security create-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH
    security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
    security unlock-keychain -p $KEYCHAIN_PASSWORD $KEYCHAIN_PATH

    ls -l $KEYCHAIN_PATH
    security show-keychain-info $KEYCHAIN_PATH

    security import $CERTIFICATE_PATH -P $P12_PASSWORD -A -t cert -f pkcs12 -k $KEYCHAIN_PATH            
    security list-keychains -d user -s $KEYCHAIN_PATH

    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles
    ls -l ~/Library/MobileDevice/Provisioning\ Profiles
```
6. Retrieve API keys or access tokens required to communicate with the target App Store
```yaml
- name: Decode App Store Connect private key file and save it
  env:
    API_KEY_BASE64: ${{ secrets.iOS_APPSTORE_CONNECT_PRIVATE_KEY_BASE64 }}
    API_KEY: ${{ secrets.iOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
    mkdir -p ~/private_keys
    ls ~/private_keys
    echo -n "$API_KEY_BASE64" | base64 --decode -o ~/private_keys/AuthKey_$API_KEY.p8
    echo "After saving: "
    ls ~/private_keys
```
7. Compile the app for the target platform
```yaml
- name: Build iOS application archive
  run: |
    echo "Listing keychains before build:"
    security list-keychains

    flutter doctor

    ls ~/Library/MobileDevice/Provisioning\ Profiles
    flutter build ipa --release --export-options-plist=ios/exportOptions.plist
```
8. Upload the compiled build to the target App Store.
```yaml
- name: Upload to App Store Connect
  env:
    ISSUER_ID: ${{ secrets.IOS_APPSTORE_CONNECT_ISSUER_ID }}          
    API_KEY: ${{ secrets.IOS_APPSTORE_CONNECT_API_KEY_ID }}     
  run: |
      echo "Before uploading: "
      ls ~/private_keys
      xcrun altool --upload-app -f build/ios/ipa/*.ipa -t ios --apiKey $API_KEY --apiIssuer "$ISSUER_ID"
      ls ~/private_keys
```
9. Remove any sensitive data, temporary files, and credentials from the CD environment to maintain security.
```yaml
- name: Clean up keychain and provisioning profile
      if: ${{ always() }}
      run: |
        security delete-keychain $RUNNER_TEMP/app-signing.keychain-db
        rm ~/Library/MobileDevice/Provisioning\ Profiles/build_pp.mobileprovision
```

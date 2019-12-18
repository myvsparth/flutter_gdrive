# Google Drive implementation in Flutter (Upload, List & Download)

## Introduction:

- In this article we will learn how to integrate Google Drive in Flutter App. We will learn how we can upload, list and download files to Google drive using Flutter. I have used Google Plus login for authentication and then accessed google drive. I have studied Google Drive Api but found this way is better and native at the end. So let’s start our Google Drive implementation in Flutter.

## Output
![alt text](https://raw.githubusercontent.com/myvsparth/flutter_gdrive/master/screenshots/1.png)

## Steps:
1. First and basic step to create new application in flutter. If you are a beginner in flutter then you can check my blog [Create a first app in Flutter](https://www.c-sharpcorner.com/blogs/create-a-first-flutter-app-in-visual-studio-code). I have created an app named as “flutter_gdrive”.

2. Integrate Google Plus Login into your project. I have wrote an article on this. Please [check it out](https://www.c-sharpcorner.com/article/how-to-do-simple-login-with-email-id-in-flutter-using-google-firebase/).

3. Now, as you have already created Google Firebase Project, now it’s time to enable Google Drive API from Google Developer Console. Note that you need to select the same project in Google Developer Console as you have created in Google Firebase. In my case  it is “Flutter Gdrive”. Check below screenshots to understand it more.

    ![alt text](https://raw.githubusercontent.com/myvsparth/flutter_gdrive/master/screenshots/2.png)

    ![alt text](https://raw.githubusercontent.com/myvsparth/flutter_gdrive/master/screenshots/3.png)

4. Now, we will add the dependencies for implementing Google Drive Operations (Google API, Pick File to Upload & Download file in mobile storage). Please check below dependencies.

    -   Add below dependencies and save the file
        ```
        dependencies:
            flutter:
                sdk: flutter
            cupertino_icons: ^0.1.2
            firebase_auth: ^0.15.2
            google_sign_in: ^4.1.0
            flutter_secure_storage: ^3.3.1+1
            googleapis: ^0.54.0
            googleapis_auth: ^0.2.11
            path_provider: ^1.5.1
            file_picker: ^1.3.8
        ```
5. We need to define permission for file reading and writing. For that define following permissions in AndroidManifest File. (android/app/src/main/AndroidManifest.xml)

    ```
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
    ```
 
6. Now, we will implement File Upload, Listing and Download Login programmatically in main.dart file. Following is the class for mapping Google Auth Credential to GoogleHttpClient Credentials.

    ```
    class GoogleHttpClient extends IOClient {
        Map<String, String> _headers;
        GoogleHttpClient(this._headers) : super();
        @override
        Future<http.StreamedResponse> send(http.BaseRequest request) =>
            super.send(request..headers.addAll(_headers));
        @override
        Future<http.Response> head(Object url, {Map<String, String> headers}) =>
            super.head(url, headers: headers..addAll(_headers));
    }
    ```
7. Following is the programming implementation of google plus login, logout, file upload to google drive, file download and listing of uploaded files on google drive.
- Google Login:
    ```
    Future<void> _loginWithGoogle() async {
        signedIn = await storage.read(key: "signedIn") == "true" ? true : false;
        googleSignIn.onCurrentUserChanged
            .listen((GoogleSignInAccount googleSignInAccount) async {
            if (googleSignInAccount != null) {
            _afterGoogleLogin(googleSignInAccount);
            }
        });
        if (signedIn) {
            try {
            googleSignIn.signInSilently().whenComplete(() => () {});
            } catch (e) {
            storage.write(key: "signedIn", value: "false").then((value) {
                setState(() {
                signedIn = false;
                });
            });
            }
        } else {
            final GoogleSignInAccount googleSignInAccount =
                await googleSignIn.signIn();
            _afterGoogleLogin(googleSignInAccount);
        }
        }
        
        Future<void> _afterGoogleLogin(GoogleSignInAccount gSA) async {
        googleSignInAccount = gSA;
        final GoogleSignInAuthentication googleSignInAuthentication =
            await googleSignInAccount.authentication;
        
        final AuthCredential credential = GoogleAuthProvider.getCredential(
            accessToken: googleSignInAuthentication.accessToken,
            idToken: googleSignInAuthentication.idToken,
        );
        
        final AuthResult authResult = await _auth.signInWithCredential(credential);
        final FirebaseUser user = authResult.user;
        
        assert(!user.isAnonymous);
        assert(await user.getIdToken() != null);
        
        final FirebaseUser currentUser = await _auth.currentUser();
        assert(user.uid == currentUser.uid);
        
        print('signInWithGoogle succeeded: $user');
        
        storage.write(key: "signedIn", value: "true").then((value) {
            setState(() {
            signedIn = true;
            });
        });
    }
    ```
- Google Logout:
    ```
    void _logoutFromGoogle() async {
        googleSignIn.signOut().then((value) {
            print("User Sign Out");
            storage.write(key: "signedIn", value: "false").then((value) {
            setState(() {
                signedIn = false;
            });
            });
        });
    }
    ```
- Upload File to Google Drive:
    ```
    _uploadFileToGoogleDrive() async {
        var client = GoogleHttpClient(await googleSignInAccount.authHeaders);
        var drive = ga.DriveApi(client);
        ga.File fileToUpload = ga.File();
        var file = await FilePicker.getFile();
        fileToUpload.parents = ["appDataFolder"];
        fileToUpload.name = path.basename(file.absolute.path);
        var response = await drive.files.create(
            fileToUpload,
            uploadMedia: ga.Media(file.openRead(), file.lengthSync()),
        );
        print(response);
    }
    ```
- List Uploaded Files to Google Drive
    ```
    Future<void> _listGoogleDriveFiles() async {
        var client = GoogleHttpClient(await googleSignInAccount.authHeaders);
        var drive = ga.DriveApi(client);
        drive.files.list(spaces: 'appDataFolder').then((value) {
            setState(() {
            list = value;
            });
            for (var i = 0; i < list.files.length; i++) {
            print("Id: ${list.files[i].id} File Name:${list.files[i].name}");
            }
        });
    }
    ```
- Download Google Drive File
    ```
    Future<void> _downloadGoogleDriveFile(String fName, String gdID) async {
        var client = GoogleHttpClient(await googleSignInAccount.authHeaders);
        var drive = ga.DriveApi(client);
        ga.Media file = await drive.files
            .get(gdID, downloadOptions: ga.DownloadOptions.FullMedia);
        print(file.stream);
        
        final directory = await getExternalStorageDirectory();
        print(directory.path);
        final saveFile = File('${directory.path}/${new DateTime.now().millisecondsSinceEpoch}$fName');
        List<int> dataStore = [];
        file.stream.listen((data) {
            print("DataReceived: ${data.length}");
            dataStore.insertAll(dataStore.length, data);
        }, onDone: () {
            print("Task Done");
            saveFile.writeAsBytes(dataStore);
            print("File saved at ${saveFile.path}");
        }, onError: (error) {
            print("Some Error");
        });
    }
    ```

8. Great we are done with Google Drive Integration in Flutter. Please check my full source code for it on Git.

## NOTE:
-   Please check source code for full example.

## Conclusion:
- We have learnt how to integrate Google Drive in Flutter and perform upload, download and list operations.

For help getting started with Flutter, view 
[online documentation](https://flutter.dev/docs), which offers tutorials, 
samples, guidance on mobile development, and a full API reference.
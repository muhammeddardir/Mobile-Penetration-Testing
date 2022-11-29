# mobile-pentest-Diva-walkthrough
This repo contain Writeup for vulnerable mobile application called DIVA
1. **Insecure logging**
- To know activity class in jadx check activity name in app and in jadx check class name then see code
-  Log.e : send error to log with my input : Note log have more Options ex: log.i 
- we can get pkg_name :  `adb shell pm list packages`
- Get Pid `adb shell ps -a` or `adb shell pidof -s pkg_name`
- we can check log by : `adb logcat --pid=2386`  or `adb logcat --pid=$(adb shell pidof -s pkg_name)`
- Mitigation : don't add input in logs
- In Real secnario Run log when I add any input in app and test when we add valid data or not and we can Open app in jadx and from tool par select Navigation then classSearch and checkbox code and write in search par `log.`
- evry app can read logs for any app
***
2. **HardcodeActivity**
- Find Secrets in code
***
**sharedpreferences folder contain xml files**
- Interface for accessing and modifying preference data returned by `Context.getSharedPreferences(String, int)` 
- Retrieve and hold the contents of the preferences file 'name', returning a SharedPreferences through which you can retrieve and modify its values
- If the preferences directory does not already exist, it will be created when this method is called.
- location : `/data/data/app_pkg_name` 

3. Insecure DataStorage Activity 1
code : 
```
SharedPreferences spref = PreferenceManager.getDefaultSharedPreferences(this); #create object
SharedPreferences.Editor spedit = spref.edit(); # Open object in editor mode as var spedit
```
- Save Input data in Preference folder
- Impact when app installed in rooted device other apps can read this files
***
4. Insecure DataStorage Activity 2
1. App create DB
2. Get user and pwd from user gui
3. insert input data user,pwd to database
4. If error occure ` Log.d("Diva", "Error occurred while inserting into database: " + e.getMessage());` :  
5. we can find database in databses folder in  `/data/data/pkg_app_name` search by DB name
6. Download databse in my device by `adb pull /data/data/pkg_app_name/databses`
7.  Mobile app is sqlite DB
8. `sqlite3 database_name`
9. `.table` : to list all table in db
10. `select * from table_name;` 
11. Mitigate : encrypt creds
***
5. Insecure DataStorage Activity 3
- Get user and pwd from user gui
- open file and write in it creds we can find file in `/data/data/pkg_app_name`
- code `File uinfo = File.createTempFile("uinfo", "tmp", ddir);`
- Create file with name : ninfo+Randon_char+tmp
- ddir : is data directory : `/data/data/pkg_app_name`
***
6.  Insecure DataStorage Activity 4
code
`File sdir = Environment.getExternalStorageDirectory();` : find sdcard location
`File uinfo = new File(sdir.getAbsolutePath() + "/.uinfo.txt");` create file in sdcard hidden file
- sdcard location in : `/mnt/sdcard` 
***
**Data storage Recommendation**
1. Asymmetric with add key in "android key store" store in container and we can communicate with api to call key or make backend handel encryption and decryption
2. If flater we can use plugin : " fliutter secure storage" that usr AES Encryption and AES key encrypt by RSA key and RSA key stored in key stored.
***
7. Input validation Issues 1
- test sqli by add `'` : and run catlog to see logs that return error by sql statment 
- `'or 1=1 --`
***
8. Input validation Issues 2
- code
```
EditText uriText = (EditText) findViewById(R.id.ivi2uri); 
WebView wview = (WebView) findViewById(R.id.ivi2wview);
wview.loadUrl(uriText.getText().toString());
```
- WebView : class let me display web pages in activities
- we can use file:///data/data/jakhar.aseem.diva/shared_prefs/jakhar.aseem.diva_preferences.xml
- if two app have same uid that mean two app can share data together called "shared uid"
***
**How to interact with activities**
- Activity be exported when be Explicit intents or define intent in activite if Activity  exported means another app can call this activity
- `adb shell am start -a ` or write app to call activity : `-a`"  pass intent to activity",  `--es` : pass string, `--ez` pass boolen, `-n`  call component
- we can doing Enforce Browsing
***
9. Access control Issues 1
1. visit api activity `APICredsActivity`
2. check api activity in mainifset.xml if exported or not in 
3. call `APICredsActivity` by adb `adb shell am start -n pkg_name/.activity_name` 
***
10. Access control Issues 2 
1. in Ativity AccesscontrolIssues2 code 
code :
- `i.setAction("jakhar.aseem.diva.action.VIEW_CREDS2");`  add action in intent in activity. and add validation in check_pin ` i.putExtra(getString(R.string.chk_pin), chk_pin);`
2. Search in mainifset.xml about `jakhar.aseem.diva.action.VIEW_CREDS2` action we notic this action  uder > intent > jakhar.aseem.diva.APICreds2Activity
3. Check code for `jakhar.aseem.diva.APICreds2Activity` activity. 
4. visit activity  `jakhar.aseem.diva.APICreds2Activity` and check code we find
```java
  boolean bcheck = i.getBooleanExtra(getString(R.string.chk_pin), true); // will check chk_pin if not found will make value true will be !Ture  = false
if (!bcheck) {
	apicview.setText("TVEETER API Key: secrettveeterapikey\nAPI User name: diva2\nAPI Password: p@ssword2");
            return;}
	apicview.setText("Register yourself at http://payatu.com to get your PIN and then login with that PIN!");
```
5. this code check bin if true will return api data
6. Exploit code 
	1. we need make bcheck = false beacuse !false = true
	2. search in strings about  `R.string.chk_pin` this mean  `chk_pin`  is called from strings we will find `chk_pin` name in strings file we notic this line                                                                      `<string name="chk_pin">check_pin</string>`
	3. `adb shell am start -n pkg_name/.activity_name --ez "check_pin" "false"` : ez send boolen data
***
11. Access control Issues 3
1. check `Access control Issues 3` activity we will find it's start activity called `AccessControl3NotesActivity` not two activities  `Access control Issues 3` and `AccessControl3NotesActivity` not exported
2. we will check `AccessControl3NotesActivity` we find in code this line `getContentResolver().query` thats mean `AccessControl3NotesActivity` call content provider
3. we will visit mainifset.xml content provider we find `jakhar.aseem.diva.NotesProvider` exported
4. we will check `jakhar.aseem.diva.NotesProvider` content provider we notic fetch data by `content://jakhar.aseem.diva.provider.notesprovider/notes`
5. Exploit                                                                                                                                   
`adb.exe shell content query --uri content://jakhar.aseem.diva.provider.notesprovider/notes`
***
12. Hardcoded issue part 2
- code in activity
```java
if (this.djni.access(hckey.getText().toString()) != 0)
```
- `access` is function in class `djni` that validate hckey and return 0,1
- in access function code :
```java
package jakhar.aseem.diva;
/* loaded from: classes.dex */
public class DivaJni {
    private static final String soName = "divajni";
    public native int access(String str);
    public native int initiateLaunchSequence(String str);
    static {
        System.loadLibrary(soName);
    }
}
``` 
- `public native int access(String str);` navtive function implement in divajni library writen by c++ language to Read `access` function we need decompile apk to read any function in native library 
- we will decompile apk by : `apktool d name.apk` visit app and go to `lib` directory "lib contain native libray that app use it"
- change directory to `x86` know we need to revers `libdivajni.so` copy library in host pc and Open it by IDA 
- In IDA visit access function in v3 we will find key that loop use it to compare by our input
- Don't add sensitive data in native library
***
13. Input validation Issue part 3
- code : in activity 
```java
if (this.djni.initiateLaunchSequence(cTxt.getText().toString()) != 0)
```

- code for `initiateLaunchSequence`
```java
package jakhar.aseem.diva;
/* loaded from: classes.dex */
public class DivaJni {
    private static final String soName = "divajni";
    public native int access(String str);
    public native int initiateLaunchSequence(String str);
    static {
        System.loadLibrary(soName);
    }
}
```
- we will search in github for  DivaJni source code we will find this line `#define CODESIZEMAX 20`
- we must add input must be more than 20 characters the array would overflow causing app to crash.
***

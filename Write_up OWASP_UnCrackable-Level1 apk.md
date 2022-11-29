**Bypass root detection**

**What is rooting**
- Rooting is a process that allows you to attain root access to the Android operating system code
- [Root Detection Method](https://github.com/OWASP/owasp-mastg/blob/master/Document/0x05j-Testing-Resiliency-Against-Reverse-Engineering.md)
***
- `MainActivity` Class 
```java
    @Override // android.app.Activity
protected void onCreate(Bundle bundle) {
	if (c.a() || c.b() || c.c()) {
        a("Root detected!");
    }
    if (b.a(getApplicationContext())) {
        a("App is debuggable!");
    }
    super.onCreate(bundle);
    setContentView(R.layout.activity_main);
}
```
- Class `c` code
```java
package sg.vantagepoint.a;

import android.os.Build;
import java.io.File;

/* loaded from: classes.dex */
public class c {
    public static boolean a() {
        for (String str : System.getenv("PATH").split(":")) {
            if (new File(str, "su").exists()) {
                return true;
            }
        }
        return false;
    }

    public static boolean b() {
        String str = Build.TAGS;
        return str != null && str.contains("test-keys");
    }

    public static boolean c() {
        for (String str : new String[]{"/system/app/Superuser.apk", "/system/xbin/daemonsu", "/system/etc/init.d/99SuperSUDaemon", "/system/bin/.ext/.su", "/system/etc/.has_su_daemon", "/system/etc/.installed_su_daemon", "/dev/com.koushikdutta.superuser.daemon/"}) {
            if (new File(str).exists()) {
                return true;
            }
        }
        return false;
    }
}
```
- **Now we need to bypass root Detection using frida script**
1. Create js file `bypassroot.js`
```javascript
java.perform(function (){
var myClass = java.use("sg.vantagepoint.a.c");//class instance
myClass.a.implementation = function(){ //edite a functiom
	return false;  
myClass.b.implementation = function(){ //edite b functiom
	return false; 
myClass.c.implementation = function(){ //edite c functiom
	return false; 
}
})
```
- Now Run frida script : `Frida-ps -U -f owasp.mstg.uncrackable1 -l bypassroot.js`

***
**Find Secret**
- After Check root Process, below it we can find function `verify` that take our input and pass it to function `a` in class `a` : `a.a`
```java
 public void verify(View view) {
        String str;
        String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
        AlertDialog create = new AlertDialog.Builder(this).create();
        if (a.a(obj)) {
            create.setTitle("Success!");
            str = "This is the correct secret.";
        } else {
            create.setTitle("Nope...");
            str = "That's not it. Try again.";
        }
```
- After we check class `a` we can find
```java
package sg.vantagepoint.uncrackable1;

import android.util.Base64;
import android.util.Log;

/* loaded from: classes.dex */
public class a {
    public static boolean a(String str) {
        byte[] bArr;
        byte[] bArr2 = new byte[0];
        try {
            bArr = sg.vantagepoint.a.a.a(b("8d127684cbc37c17616d806cf50473cc"), Base64.decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0));
        } catch (Exception e) {
            Log.d("CodeCheck", "AES error:" + e.getMessage());
            bArr = bArr2;
        }
        return str.equals(new String(bArr));
    }

    public static byte[] b(String str) {
        int length = str.length();
        byte[] bArr = new byte[length / 2];
        for (int i = 0; i < length; i += 2) {
            bArr[i / 2] = (byte) ((Character.digit(str.charAt(i), 16) << 4) + Character.digit(str.charAt(i + 1), 16));
        }
        return bArr;
    }
}
```
- Class `a` inside it function `a` that take string and compare this string by byte array : `bArr` in this line `return str.equals(new String(bArr));` if True is success
- This mean `bArr` is secret string contain this function `sg.vantagepoint.a.a.a(...)`  this function take two parameter first parmeter call function `b` and pass for it this value `"8d127684cbc37c17616d806cf50473cc"`, Second Parameter is base64 decode : `5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=`
- And Function `b` to convert from hex to ascii
- **Note** : this function `sg.vantagepoint.a.a.a` process two input and pass it to `bArr` take two parameters and pass `bArr` to function `b` to Decode it and compare by my input string
- We will check function `sg.vantagepoint.a.a.a` 
```java
public class a {
    public static byte[] a(byte[] bArr, byte[] bArr2) {
        SecretKeySpec secretKeySpec = new SecretKeySpec(bArr, "AES/ECB/PKCS7Padding");
        Cipher cipher = Cipher.getInstance("AES");
        cipher.init(2, secretKeySpec);
        return cipher.doFinal(bArr2);
    }
}
```
- this code take parameter1 and Parameter2 as show in code : `public static byte[] a(byte[] bArr, byte[] bArr2)`
- Then last line ` return cipher.doFinal(bArr2);` return base64 decode by this line we can understand that base64 string is a secret 
- Encryption type : `AES` with mode `ECB`
- chiper text is  : `5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=`
- and init secret key by parameter1 : `8d127684cbc37c17616d806cf50473cc`
- Using Cyberchif we Get Secret: `I want to believe`






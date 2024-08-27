---
layout: post
title: Cheat Sheet Android 
---

![]({{ site.baseurl }}/images/android-logo.png){:class="img-responsive"}

# Android Pentest 



## Port forwarding 

```bash
# clé de connexion console 
cat .emulator_console_auth_token 
BfdYTU4khvAHJWYZ
# console 
telnet localhost 5554
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Android Console: Authentication required
Android Console: type 'auth <auth_token>' to authenticate
Android Console: you can find your <auth_token> in 
'/home/jugu-ubuntu/.emulator_console_auth_token'
# authent 
auth BfdYTU4khvAHJWYZ
Android Console: type 'help' for a list of commands
OK
# création du forwarding 
redir add tcp:6666:5555
OK
# le port ecoute en local sur l'hote 
ss -laputen | grep 6666
tcp   LISTEN     0      1                               127.0.0.1:6666             0.0.0.0:*     users:(("qemu-system-x86",pid=18861,fd=157)) uid:1000 ino:387832 sk:2006 cgroup:/user.slice/user-1000.slice/user@1000.service/app.slice/app-gnome-android\x2dstudio_android\x2dstudio-16685.scope <->
# Depuis la kali : 
ssh -L 0.0.0.0:5555:127.0.0.1:6666 jugu-ubuntu@192.168.100.1
# enfin ADB marche : 
adb connect 127.0.0.1
connected to 127.0.0.1:5555
adb devices          
List of devices attached
127.0.0.1:5555  device
# Pousser un APK : 
adb -s 127.0.0.1:5555 install Downloads/AndroGoat.apk # -s requis si plusieurs devices 


```



## Frida

Il faut télécharger le serveur Frida ici : 
https://github.com/frida/frida/releases

> Note: Dans le cas de android studio, prendre une build x86 6a bit

```bash
# passage en mode root 
adb root
# connection au smartphone 
adb connect 10.0.2.2:5555
# upload du serveur frida 
adb push '/home/kali/Tools/frida-server-16.4.3-android-x86_64'  /data/local/tmp/frida-server 
# vérifier que le serveur est bien sur le device : 
adb shell "ls -al /data/local/tmp "                           
total 110744
drwxrwx--x 2 shell shell      4096 2024-07-16 07:57 .
drwxr-x--x 5 root  root       4096 2024-07-15 09:10 ..
-rwxrwxrwx 1 root  root  113380152 2024-07-15 14:02 frida-server
```

Puis lancer un shell avec `adb shell` : 

```bash
# rendre executable le serveur
chmod 755 /data/local/tmp/frida-server 
# lancer le serveur en arrière plan 
/data/local/tmp/frida-server & 
```

> Si le serveur est déjà en écoute : 

```bash
emulator64_x86_64:/ # Unable to start: Error binding to address 127.0.0.1:27042: Address already in use
# lancement ADB shell
adb shell 
# On cherche le process 
emulator64_x86_64:/ # ss -laputen | grep frida-server                                                             
tcp    LISTEN     0      10     127.0.0.1:27042              0.0.0.0:*                   users:(("frida-server",pid=3251,fd=8)) ino:62781 sk:308a <->
# on le kill
kill -9 3251
/data/local/tmp/frida-server & 
```

Une fois le serveur lancé on cherche le package a pentest : 

```bash
adb shell pm list packages | grep owasp
package:owasp.sat.agoat
```

Si erreur : 

```bash
adb root                    
adb: unable to connect for root: more than one device/emulator
```

Utiliser `-s` : 

```bash
adb -s 127.0.0.1:5555 root  
restarting adbd as root
```



## Objection 

Maintenant place au bypass : 

```bash
objection -g owasp.sat.agoat explore 
Using USB device `Android SDK built for x86 64`
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
owasp.sat.agoat on (Android: 13) [usb] # android sslpinning disable
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check$okhttp()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 308548. Type: android-sslpinning-disable

```

Pour préciser un device précis : 

```bash
#recup de l'ID du device 
frida-ls-devices
Id             Type    Name                          OS                   
-------------  ------  ----------------------------  ---------------------
local          local   kali                          Kali GNU/Linux 2023.4
10.0.2.2:5555  usb     Android SDK built for x86 64  Android 13           
barebone       remote  GDB Remote Stub                                    
socket         remote  Local Socket     

# lancement d'objection 
objection -S 10.0.2.2:5555 -g owasp.sat.agoat explore
Using USB device `Android SDK built for x86 64`
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
owasp.sat.agoat on (Android: 13) [usb] # android sslpinning disable
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check()
(agent) Found okhttp3.CertificatePinner, overriding CertificatePinner.check$okhttp()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job 848185. Type: android-sslpinning-disable
owasp.sat.agoat on (Android: 13) [usb] # (agent) [848185] Called SSLContext.init(), overriding TrustManager with empty one.
(agent) [848185] Called OkHTTP 3.x CertificatePinner.check(), not throwing an exception.
```

## Unprotected android component 

### Les attribut du manifest : 

L'attribut `android:exported` indique si un composant (activité, service, broadcast receiver, etc.) peut être lancé par des composants d'autres applications :

Si la valeur est true, n'importe quelle application peut accéder à l'activité et la lancer avec le nom exact de la classe.
Si la valeur est false, seuls les composants de la même application, les applications ayant le même ID utilisateur ou les composants système privilégiés peuvent lancer l'activité.


### les activity 

Une activité (activity) représente un écran dans une application Android.

Dans le code se trouve une référence au PIN : 

```java
if (accessControlIssue1Activity.isPinCorrect(pinValue.getText().toString())) {
                    Toast.makeText(AccessControlIssue1Activity.this.getApplicationContext(), "PIN Verified", 1).show();
                    Intent myIntent = new Intent(AccessControlIssue1Activity.this, (Class<?>) AccessControl1ViewActivity.class);
                    AccessControlIssue1Activity.this.startActivity(myIntent);
                    return;
                }
```

L'activité `AccessControl1ViewActivity` est lancé si le code pin est juste. 
Lançons cette dernière : 

```bash
adb shell                              
255|emulator64_x86_64:/ # am start owasp.sat.agoat/.AccessControl1ViewActivity                                    
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=owasp.sat.agoat/.AccessControl1ViewActivity }
```

Cela bypass complètement l'insertion du PIN.

### Custom URL scheme 

C'est une option de configuration avancée utilisée pour définir un format de lien non standard qui s'ouvrira uniquement dans votre application et non dans le navigateur de l'appareil, par exemple. toto://votresite.com/path.

Dans le manifest se trouve ceci : 

```xml
        <activity
            android:label="@string/activity"
            android:name="owasp.sat.agoat.AccessControl1ViewActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data
                    android:scheme="androgoat"
                    android:host="vulnapp"/>
            </intent-filter>
        </activity>
```

Il est donc possible d'appeler l'activité `AccessControl1ViewActivity` via un appel avec `androgoat://vulnapp`. Cela se confirme via ADB : 

```bash
dumpsys package owasp.sat.agoat              
Activity Resolver Table:
  Schemes:
      androgoat:
        900b353 owasp.sat.agoat/.AccessControl1ViewActivity filter 121d490
          Action: "android.intent.action.VIEW"
          Category: "android.intent.category.DEFAULT"
          Scheme: "androgoat"
          Authority: "vulnapp": -1

```

Pour appeler ceci : 

```bash
emulator64_x86_64:/ # am start -d "androgoat://vulnapp"                                                           
Starting: Intent { dat=androgoat://vulnapp/... }
```

### Les services 

Un service est un composant Android qui s’exécute sans interface graphique. Il dispose de son propre cycle de vie géré par le système. Un service est idéal pour implémenter un traitement long pour lequel on ne souhaite pas bloquer l’activité de l’utilisateur. Par exemple, une application peut avoir besoin d’envoyer en masse des données à un serveur.

Je trouve un service au nom sympa : 

```xml
        <service
            android:name="owasp.sat.agoat.DownloadInvoiceService"
            android:enabled="true"
            android:exported="true"/>
```

Il semble ici possible de trigger ce service pour télécharger la facture : 

```bash
emulator64_x86_64:/ # am start-foreground-service owasp.sat.agoat/.DownloadInvoiceService
Starting service: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=owasp.sat.agoat/.DownloadInvoiceService }
```

### Broadcast receiver : 

Le BroadcastReceiver est une classe abstraite qui permet de créer un composant en charge de recevoir des broadcasts.

Dans le manifest se trouve un receiver : 

```xml
        <receiver
            android:name="owasp.sat.agoat.ShowDataReceiver"
            android:enabled="true"
            android:exported="true"/>
```

Il faut le trigger, voici son code : 

```java 
public final class ShowDataReceiver extends BroadcastReceiver {
    @Override // android.content.BroadcastReceiver
    public void onReceive(Context context, Intent intent) {
        Intrinsics.checkParameterIsNotNull(context, "context");
        Intrinsics.checkParameterIsNotNull(intent, "intent");
        Toast.makeText(context, "Username is CrazyUser, Password is CrazyPassword and Key is 123myKey456", 1).show();
    }
}
```

Objection le voit aussi : 

```bash
owasp.sat.agoat on (Android: 13) [usb] # android hooking list receivers
owasp.sat.agoat.ShowDataReceiver

Found 1 classes

```

Il faut ensuite le trigger via son action : 

```bash
am broadcast -n owasp.sat.agoat/.ShowDataReceiver
```

## Lire des logs 

```bash
emulator64_x86_64:/ # pidof owasp.sat.agoat
2953
1|emulator64_x86_64:/ # logcat --pid=2953 
```

Pour nettoyer les logs : 

```bash
logcat --pid=2953 | grep -v "EGL_emulation"
```

## Insecure Data Storage 

### shared preferences 

Si vous souhaitez enregistrer une collection de clés/valeurs dont la taille est relativement petite, vous pouvez utiliser les API SharedPreferences. Un objet SharedPreferences renvoie vers un fichier contenant des paires clé/valeur et fournit des méthodes simples pour les lire et les écrire. Chaque fichier SharedPreferences est géré par le framework. Il peut être privé ou partagé.

En lisant le code je trouve le bout de code suivant : 

```java
public final void onClick(View it) {
                SharedPreferences sharedPreference = InsecureStorageSharedPrefs.this.getSharedPreferences("users", 0);
                SharedPreferences.Editor editor = sharedPreference.edit();
                EditText username2 = username;
                Intrinsics.checkExpressionValueIsNotNull(username2, "username");
                editor.putString("username", username2.getText().toString());
                EditText password2 = password;
                Intrinsics.checkExpressionValueIsNotNull(password2, "password");
                editor.putString("password", password2.getText().toString());
                if (editor.commit()) {
                    Toast.makeText(InsecureStorageSharedPrefs.this.getApplicationContext(), "Data saved", 1).show();
                } else {
                    Toast.makeText(InsecureStorageSharedPrefs.this.getApplicationContext(), "Data not saved", 1).show();
                }
            }
```

Ici il semble qu'un fichier `users.xml` va être crée :

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="password">test</string>
    <string name="username">test</string>
</map>
```

> Note: les fichiers portent la convention de nommage suivante : `/data/data/<package-name>/shared_prefs/<nom_pref>.xml`

Il est également possible de les éditer. Par exemple prenons ce code de gestion de score dans une jeu : 

```java 
public final int getScoreFromSP() {
        int Score = 0;
        int Level = 1;
        SharedPreferences sharedPreferences = getSharedPreferences("score", 0);
        if (sharedPreferences.getInt("score", 0) != 0 && sharedPreferences.getInt("level", 0) != 0) {
            Score = sharedPreferences.getInt("score", 0);
            Level = sharedPreferences.getInt("level", 0);
        }
        System.out.println((Object) ("Score is " + Score + " and Level is" + Level));
        return Score;
    }
```

Il est possible d'afficher le score : 

```xml
# cat /data/data/owasp.sat.agoat/shared_prefs/score.xml                                       
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="score" value="1" />
    <int name="level" value="1" />
</map>

```

Puis de le modifier : 

```bash
emulator64_x86_64:/ # sed -i 's/name=\"score\"\ value=\"10004\"/name=\"score\"\ value=\"1077777\"/g' owasp.sat.agoat/shared_prefs/score.xml
emulator64_x86_64:/ # cat /data/data/owasp.sat.agoat/shared_prefs/score.xml                                       
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="score" value="1077777" />
    <int name="level" value="2" />
</map>
```

### SQLite 

Voici un exemple de code vulnérable : 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        try {
            SQLiteDatabase openOrCreateDatabase = openOrCreateDatabase("aGoat", 0, null);
            this.mDB = openOrCreateDatabase;
            if (openOrCreateDatabase == null) {
                Intrinsics.throwNpe();
            }
```

Si le code ne hash pas les mots de passe il deviens alors possible de récupérer ces derniers avec un explorateur de base SQlite. 

> Note : le fichier se trouve ici : `/data/data/owasp.sat.agoat/databases/aGoat`

On le récupère ainsi : 

```bash
adb pull /data/data/owasp.sat.agoat/databases/aGoat
```

### Fichiers temporaires 

```java
            public final void onClick(View it) {
                try {
                    File userinfo = File.createTempFile("users", "tmp", new File(InsecureStorageTempActivity.this.getApplicationInfo().dataDir));
                    userinfo.setReadable(true);
                    userinfo.setWritable(true);
                    FileWriter fw = new FileWriter(userinfo);
                    StringBuilder sb = new StringBuilder();
```

Ceci crée un fichier `/data/data/owasp.sat.agoat/users7736633838409041098tmp` : 

```bash
cat /data/data/owasp.sat.agoat/users7736633838409041098tmp
username is toto
password is toto
```


### Carte SD 

Il est possible de stocker des temporaire de la même manière dans la carte SD : 

```java
public final void onClick(View it) {
                try {
                    StringBuilder sb = new StringBuilder();
                    sb.append(" This data is stored in SdCard on ");
                    sb.append(new Date());
                    sb.append(": \n Username - ");
                    EditText username2 = username;
                    Intrinsics.checkExpressionValueIsNotNull(username2, "username");
                    sb.append(username2.getText().toString());
                    sb.append(" Password -");
                    EditText password2 = password;
                    Intrinsics.checkExpressionValueIsNotNull(password2, "password");
                    sb.append(password2.getText().toString());
                    sb.append("\n");
                    String data = sb.toString();
                    File externalStorageDirectory = Environment.getExternalStorageDirectory();
                    Intrinsics.checkExpressionValueIsNotNull(externalStorageDirectory, "Environment.getExternalStorageDirectory()");
                    File userinfo = File.createTempFile("users", "tmp", new File(externalStorageDirectory.getAbsolutePath()));
                    System.out.println((Object) ("userinfo " + userinfo));
                    userinfo.setReadable(true);
                    userinfo.setWritable(true);
                    FileWriter fw = new FileWriter(userinfo);
                    fw.write(data);
                    fw.close();
                    Toast.makeText(InsecureStorageSDCardActivity.this.getApplicationContext(), "Data saved", 1).show();
                } catch (IOException e) {
                    e.printStackTrace();
                }
```

Et il est affichable de la même manière : 

```bash
emulator64_x86_64:/ $ ls /sdcard/                                                                                 
Alarms   Audiobooks  Documents  Movies  Notifications  Podcasts    Ringtones
Android  DCIM        Download   Music   Pictures       Recordings  users262239251926958618tmp
emulator64_x86_64:/ $ cat /sdcard/users262239251926958618tmp
 This data is stored in SdCard on Wed Jul 17 13:42:15 GMT 2024: 
 Username - toto Password -toto
```

## Input validation 

### XSS 

Simple comme bonjour, le payload est le suivant : 

```html
<script>alert('test')</script>
```

Le code vulnérable depuis jadX montre la création d'une webview qui exploite des entrées sans filtrage  :

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_xss);
        WebView webview1 = (WebView) findViewById(R.id.webview);
        Intrinsics.checkExpressionValueIsNotNull(webview1, "webview1");
        webview1.setWebChromeClient(new WebChromeClient());
        WebSettings webSettings1 = webview1.getSettings();
        Intrinsics.checkExpressionValueIsNotNull(webSettings1, "webSettings1");
        webSettings1.setJavaScriptEnabled(true);
        webview1.loadData("<html>\n<body>\n<script>\nfunction displayContent()\n{\nvar a=document.getElementById(\"name\");\ndocument.write(a.value);\n\n}\n</script>\nName: <input type=\"text\" id=\"name\"/>\n</br></br><input type=\"button\" value=\"Display\" onclick=\"displayContent()\" style=\"background-color:black;color: white;\n  border: 2px solid #000000\"/>\n</body>\n\n</html>", "text/html", "UTF-8");
 }
```

Ce code donne ceci en HTML :

```html
<html>
    <script>
        function displayContent(){
            a=document.getElementById("name");
            document.write(a.value);
        }
    </script>

    <body>
        Name: <input type="text" id="name"/>
        </br></br><input type="button" value="Display" onclick="displayContent()"style="background-color:black;color: white; border: 2px solid #000000"/>
    </body>
</html>
```

### SQL injection : 

Ici l'insertion d'une quote suffit a trigger l'injection. Voici le code responsable :

```java
public final void onClick(View it) {
                SQLiteDatabase sQLiteDatabase;
                Cursor QryResult;
                StringBuilder strb;
                StringBuilder sb = new StringBuilder();
                sb.append("SELECT * FROM users WHERE username='");
                EditText username2 = username;
                Intrinsics.checkExpressionValueIsNotNull(username2, "username");
                sb.append(username2.getText().toString());
                sb.append("'");
                String qry = sb.toString();
```

Le code prends la valeur saisie dans le champ `username` et le pousse directement dans une requête SQL comme celle-ci : 

```SQL
SELECT * FROM users WHERE username='test'; 
```

Cette requête est injectable : 

```sql
SELECT * FROM users WHERE username='1' or '1'='1';
```

Pour cela il faut injecter `'1' or '1'='1`. 

### WebView

Le webview ne comporte pas de configuration spécifique dans le manifest. Dans le code je constate que le webview est ouvert sans fournir aucun filtrage en exploitant la valeur de l'entrée `url` : 

```java
public final void onClick(View it) {
                WebSettings webSettings1 = webview11.getSettings();
                Intrinsics.checkExpressionValueIsNotNull(webSettings1, "webSettings1");
                webSettings1.setJavaScriptEnabled(true);
                TextView url2 = url;
                Intrinsics.checkExpressionValueIsNotNull(url2, "url");
                webview11.loadUrl(url2.getText().toString());
            }
```

Il deviens alors possible de lire des fichier en saisissant simplement son chemin, par exemple `file:///data/data/owasp.sat.agoat/shared_prefs/score.xml` affiche les scores écris dans les cas précédents. 

Le code doit inclure `setAllowFileAccess(FALSE)` pour bloquer la lecture de fichier dans un WebView.        

##  Side Channel Data leakage 

### Keyboard cache 

Ici le cache du clavier permets de voler des identifiants stockés. Le code indique ici qu'aucune sécurité n'est en place pour prevenir le caching des identifiants : 

```java     
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_keyboard_cache);
        ((Button) _$_findCachedViewById(R.id.Logging1)).setOnClickListener(new View.OnClickListener() { // from class: owasp.sat.agoat.KeyboardCacheActivity$onCreate$1
            @Override // android.view.View.OnClickListener
            public final void onClick(View it) {
                Toast.makeText(KeyboardCacheActivity.this.getApplicationContext(), "Please wait while verifying your credentials ", 1).show();
            }
        });
```

Ansi il est possible de récupérer les identifiants dans `/data/data/com.google.android.inputmethod.latin/files/personal/userhistory` 

> note : sur mon Android 13 cela ne fonctionne pas.. 

### Insecure logging 

Le code suivant montre que les logs Fuient des identifiants en claire : 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_insecure_logging);
        final EditText username = (EditText) findViewById(R.id.userName);
        final EditText password = (EditText) findViewById(R.id.password);
        ((Button) _$_findCachedViewById(R.id.Logging1)).setOnClickListener(new View.OnClickListener() { // from class: owasp.sat.agoat.InsecureLoggingActivity$onCreate$1
            @Override // android.view.View.OnClickListener
            public final void onClick(View it) {
                StringBuilder sb = new StringBuilder();
                sb.append("Error occured when processing Username ");
                EditText username2 = username;
                Intrinsics.checkExpressionValueIsNotNull(username2, "username");
                sb.append((Object) username2.getText());
                sb.append("  and Password ");
                EditText password2 = password;
                Intrinsics.checkExpressionValueIsNotNull(password2, "password");
                sb.append((Object) password2.getText());
                Log.e("Error:", sb.toString());
                PrintStream printStream = System.out;
                StringBuilder sb2 = new StringBuilder();
                sb2.append("Error: Error occured when processing Username ");
                EditText username3 = username;
                Intrinsics.checkExpressionValueIsNotNull(username3, "username");
                sb2.append((Object) username3.getText());
                sb2.append("  and Password ");
                EditText password3 = password;
                Intrinsics.checkExpressionValueIsNotNull(password3, "password");
                sb2.append((Object) password3.getText());
                printStream.println(sb2.toString());
                Toast.makeText(InsecureLoggingActivity.this.getApplicationContext(), "Error Occured", 1).show();
            }
        });
```

Grace a logcat on trouve des datas : 

```bash
emulator64_arm64:/ # logcat --pid=2594 | grep -v "EGL_emulation"
08-20 13:39:00.121  2594  2594 W OnBackInvokedCallback: OnBackInvokedCallback is not enabled for the application.
08-20 13:39:00.121  2594  2594 W OnBackInvokedCallback: Set 'android:enableOnBackInvokedCallback="true"' in the application manifest.
08-20 13:39:00.146  2594  2594 D InsetsController: show(ime(), fromIme=true)
08-20 13:39:01.048  2594  2594 E Error:  : Error occured when processing Username test  and Password test
08-20 13:39:01.048  2594  2594 I System.out: Error: Error occured when processing Username test  and Password test
08-20 13:39:01.106  2594  2620 W Parcel  : Expecting binder but got null!
08-20 13:39:02.930  2594  2594 E Error:  : Error occured when processing Username test  and Password test
08-20 13:39:02.930  2594  2594 I System.out: Error: Error occured when processing Username test  and Password test
08-20 13:39:04.589  2594  2620 W Parcel  : Expecting binder but got null!
08-20 13:41:44.652  2594  2594 E Error:  : Error occured when processing Username test  and Password test
08-20 13:41:44.653  2594  2594 I System.out: Error: Error occured when processing Username test  and Password test
08-20 13:41:44.714  2594  2620 W Parcel  : Expecting binder but got null!

```

### Clipboard 

Il est possible d'éxfiltrer de la data via le clipboard : 

```java
public final void onClick(View it) {
                EditText ccValue2 = ccValue;
                Intrinsics.checkExpressionValueIsNotNull(ccValue2, "ccValue");
                String obj = ccValue2.getText().toString();
                if (obj == null) {
                    throw new TypeCastException("null cannot be cast to non-null type kotlin.CharSequence");
                }
                String ccValue1 = StringsKt.trim((CharSequence) obj).toString();
                if (ccValue1 != null) {
                    if (!(ccValue1.length() == 0)) {
                        int otp = ((Number) CollectionsKt.first(CollectionsKt.shuffled(new IntRange(1000, 9999)))).intValue();
                        Object systemService = ClipboardActivity.this.getSystemService("clipboard");
                        if (systemService == null) {
                            throw new TypeCastException("null cannot be cast to non-null type android.content.ClipboardManager");
                        }
                        ClipboardManager clipboard = (ClipboardManager) systemService;
                        ClipData clip = ClipData.newPlainText("CC Card", String.valueOf(otp));
                        clipboard.setPrimaryClip(clip);
                        Toast.makeText(ClipboardActivity.this, "OTP Generated and Copied: " + otp, 1).show();
                        return;
                    }
                }
                Toast.makeText(ClipboardActivity.this, "Credit Card shouldn't be blank", 1).show();
            }
```

Il est possible de vider le clipboard avec une commande ADB : 

```bash
input keyevent 27
```

Cela colle la valeur dans un champ. Sinon objection : 

```bash 
owasp.sat.agoat on (Android: 13) [usb] # android clipboard monitor
(agent) Warning! This module is still broken. A pull request fixing it would be awesome!
owasp.sat.agoat on (Android: 13) [usb] # (agent) [pasteboard-monitor] Data: https://github.com/hojatsajadinia/AndRoPass
(agent) [pasteboard-monitor] Data: 9399
```



## Hardcoded value 

Ici le code contiens un code promo : 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_hard_code);
        Button VerifyButton = (Button) findViewById(R.id.hardcode1);
        final TextView priceValue = (TextView) findViewById(R.id.price);
        final EditText promoCodeValue = (EditText) findViewById(R.id.promocode);
        final Ref.ObjectRef promoCode = new Ref.ObjectRef();
        promoCode.element = "NEW2019";
        VerifyButton.setOnClickListener(new View.OnClickListener() { // from class: owasp.sat.agoat.HardCodeActivity$onCreate$1
            @Override // android.view.View.OnClickListener
```

## Root detection 

Avec Objection il deviens simple de bypass la détection : 

```bash
objection -g owasp.sat.agoat explore                                  
Checking for a newer version of objection...
Using USB device `Android SDK built for arm64`
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
owasp.sat.agoat on (Android: 13) [usb] # android root simulate
(agent) Registering job 670101. Type: root-detection-enable
owasp.sat.agoat on (Android: 13) [usb] # (agent) [670101] File existence check for /system/app/Superuser.apk detected, marking as true.
owasp.sat.agoat on (Android: 13) [usb] # 
owasp.sat.agoat on (Android: 13) [usb] # android root disable
(agent) Registering job 007705. Type: root-detection-disable
owasp.sat.agoat on (Android: 13) [usb] # (agent) [007705] File existence check for /system/app/Superuser.apk detected, marking as false.
(agent) [007705] File existence check for /sbin/su detected, marking as false.
(agent) [007705] File existence check for /system/bin/su detected, marking as false.
(agent) [007705] File existence check for /system/xbin/su detected, marking as false.
(agent) [007705] File existence check for /data/local/xbin/su detected, marking as false.
(agent) [007705] File existence check for /data/local/bin/su detected, marking as false.
(agent) [007705] File existence check for /system/sd/xbin/su detected, marking as false.
(agent) [007705] File existence check for /system/bin/failsafe/su detected, marking as false.
(agent) [007705] File existence check for /data/local/su detected, marking as false.
```

Autre suggestion : https://github.com/hojatsajadinia/AndRoPass 



## Emulation detection 

Le script en annexe fait un bon job : 

```bash
frida -U -f <com.saucetomate.APPname> -l frida.js
```

Sinon depuis Objection et a la main : 

```bash
## Chopper le nom de l'activité 
owasp.sat.agoat on (Android: 13) [usb] # android hooking get current_activity
Activity: owasp.sat.agoat.EmulatorDetectionActivity
Fragment: android.arch.lifecycle.ReportFragment

## lister les methodes 
owasp.sat.agoat on (Android: 13) [usb] # android hooking list class_methods owasp.sat.agoat.EmulatorDetectionActivity
protected void owasp.sat.agoat.EmulatorDetectionActivity.onCreate(android.os.Bundle)
public android.view.View owasp.sat.agoat.EmulatorDetectionActivity._$_findCachedViewById(int)
public final boolean owasp.sat.agoat.EmulatorDetectionActivity.isEmulator()
public void owasp.sat.agoat.EmulatorDetectionActivity._$_clearFindViewByIdCache()

Found 4 method(s)

## Changer la valeur de retour par true
owasp.sat.agoat on (Android: 13) [usb] # android hooking set return_value owasp.sat.agoat.EmulatorDetectionActivity.isEmulator false
(agent) Attempting to modify return value for class owasp.sat.agoat.EmulatorDetectionActivity and method isEmulator.
(agent) Hooking owasp.sat.agoat.EmulatorDetectionActivity.isEmulator()
(agent) Registering job 152351. Type: set-return for: owasp.sat.agoat.EmulatorDetectionActivity.isEmulator
## tester
owasp.sat.agoat on (Android: 13) [usb] # (agent) [152351] Return value was not false, setting to false.

```

Conformément au code source suivant : 

```java
public final boolean isEmulator() {
        String str = Build.FINGERPRINT + Build.DEVICE + Build.MODEL + Build.BRAND + Build.PRODUCT + Build.MANUFACTURER + Build.HARDWARE;
        if (str == null) {
            throw new TypeCastException("null cannot be cast to non-null type java.lang.String");
        }
        String builddtls = str.toLowerCase();
        Intrinsics.checkExpressionValueIsNotNull(builddtls, "(this as java.lang.String).toLowerCase()");
        return StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "generic", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) EnvironmentCompat.MEDIA_UNKNOWN, false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "emulator", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "sdk", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "vbox", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "genymotion", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "x86", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "goldfish", false, 2, (Object) null) || StringsKt.contains$default((CharSequence) builddtls, (CharSequence) "test-keys", false, 2, (Object) null);
    }
```

## APK patching 

Dans ce cas de figure nous sommes face au probleme suivant : 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_binary_patching);
        if (this.isAdmin) {
            TextView isAdminText = (TextView) _$_findCachedViewById(R.id.isAdminText);
            Intrinsics.checkExpressionValueIsNotNull(isAdminText, "isAdminText");
            isAdminText.setText("You are Admin Now");
            Button adminButton = (Button) _$_findCachedViewById(R.id.adminButton);
            Intrinsics.checkExpressionValueIsNotNull(adminButton, "adminButton");
            adminButton.setEnabled(true);
        }
        ((Button) _$_findCachedViewById(R.id.adminButton)).setOnClickListener(new View.OnClickListener() { // from class: owasp.sat.agoat.BinaryPatchingActivity$onCreate$1
            @Override // android.view.View.OnClickListener
            public final void onClick(View it) {
                Toast.makeText(BinaryPatchingActivity.this, "You clicked on Administartion button", 1).show();
            }
        });
    }
```

Ici, une interface d'admin est bloqué. J'ai donc tenté de jouer avec objection pour aller changer la valeur de `isAdmin` : 

```bash
## chopper l'activité active
owasp.sat.agoat on (Android: 13) [usb] # android hooking get current_activity
Activity: owasp.sat.agoat.BinaryPatchingActivity
Fragment: android.arch.lifecycle.ReportFragment

## Lister les methodes 
owasp.sat.agoat on (Android: 13) [usb] # android hooking list class_methods owasp.sat.agoat.BinaryPatchingActivity
protected void owasp.sat.agoat.BinaryPatchingActivity.onCreate(android.os.Bundle)
public android.view.View owasp.sat.agoat.BinaryPatchingActivity._$_findCachedViewById(int)
public final boolean owasp.sat.agoat.BinaryPatchingActivity.isAdmin()
public void owasp.sat.agoat.BinaryPatchingActivity._$_clearFindViewByIdCache()

Found 4 method(s)

## Patch de la valeur de retour de IsAdmin
owasp.sat.agoat on (Android: 13) [usb] # android hooking set return_value owasp.sat.agoat.BinaryPatchingActivity.isAdmin True
(agent) Attempting to modify return value for class owasp.sat.agoat.BinaryPatchingActivity and method isAdmin.
(agent) Hooking owasp.sat.agoat.BinaryPatchingActivity.isAdmin()
(agent) Registering job 438262. Type: set-return for: owasp.sat.agoat.BinaryPatchingActivity.isAdmin

## Rechargement de l'activité
owasp.sat.agoat on (Android: 13) [usb] # android intent launch_activity owasp.sat.agoat.BinaryPatchingActivity
(agent) Starting activity owasp.sat.agoat.BinaryPatchingActivity...
(agent) Activity successfully asked to start.
```

Ici, il semble que les check sont réalisés en avance du lancement avec "oncreate". Il faut donc patcher le binaire 

## Patching 

Avec Apklabs, installer les dépendance en premier lieu : 

```bash
sudo apt install binfmt-support qemu-user-static 
```

Si erreur zipalign désinstaller le : 

```bash
sudo apt --purge remove zipalign
```

Chopper ici la version corresondant a la version de Kali : https://pkgs.org/download/zipalign?ref=linuxtldr.com

Puis la réinstaller : 

```bash
https://pkgs.org/download/zipalign?ref=linuxtldr.com
```

> note : test avec `zipalign` si l'outil affiche l'aide, c'est ok

Ensuite dans le code j'identifie la partie a patcher :

```assembly
    .line 16
    iget-boolean v0, p0, Lowasp/sat/agoat/BinaryPatchingActivity;->isAdmin:Z

    const/4 v1, 0x1

    if-ne v0, v1, :cond_0 ;;Ligne intéréssante 

    .line 18
    sget v0, Lowasp/sat/agoat/R$id;->isAdminText:I
```

Ici je vais remplacer `if-ne` par `if-eq` puis recompiler l'APK. Ainsi le code deviens le suivant : 

```java
public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_binary_patching);
        if (!this.isAdmin) {
            TextView isAdminText = (TextView) _$_findCachedViewById(R.id.isAdminText);
            Intrinsics.checkExpressionValueIsNotNull(isAdminText, "isAdminText");
            isAdminText.setText("You are Admin Now");
            Button adminButton = (Button) _$_findCachedViewById(R.id.adminButton);
            Intrinsics.checkExpressionValueIsNotNull(adminButton, "adminButton");
            adminButton.setEnabled(true);
        }
```

A l'ouverture de l'application le mode admin est accessible. 

## Script frida SSL bypass + root + emulation  

Lancement : 

```bash
frida -U -f <com.saucetomate.APPname> -l frida.js
```

Script : 

```js
const commonPaths = [
    "/data/local/bin/su",
    "/data/local/su",
    "/data/local/xbin/su",
    "/dev/com.koushikdutta.superuser.daemon/",
    "/sbin/su",
    "/system/app/Superuser.apk",
    "/system/bin/failsafe/su",
    "/system/bin/su",
    "/su/bin/su",
    "/system/etc/init.d/99SuperSUDaemon",
    "/system/sd/xbin/su",
    "/system/xbin/busybox",
    "/system/xbin/daemonsu",
    "/system/xbin/su",
    "/system/sbin/su",
    "/vendor/bin/su",
    "/cache/su",
    "/data/su",
    "/dev/su",
    "/system/bin/.ext/su",
    "/system/usr/we-need-root/su",
    "/system/app/Kinguser.apk",
    "/data/adb/magisk",
    "/sbin/.magisk",
    "/cache/.disable_magisk",
    "/dev/.magisk.unblock",
    "/cache/magisk.log",
    "/data/adb/magisk.img",
    "/data/adb/magisk.db",
    "/data/adb/magisk_simple",
    "/init.magisk.rc",
    "/system/xbin/ku.sud",
    "/data/adb/ksu",
    "/data/adb/ksud",
];

const ROOTmanagementApp = [
    "com.noshufou.android.su",
    "com.noshufou.android.su.elite",
    "eu.chainfire.supersu",
    "com.koushikdutta.superuser",
    "com.thirdparty.superuser",
    "com.yellowes.su",
    "com.koushikdutta.rommanager",
    "com.koushikdutta.rommanager.license",
    "com.dimonvideo.luckypatcher",
    "com.chelpus.lackypatch",
    "com.ramdroid.appquarantine",
    "com.ramdroid.appquarantinepro",
    "com.topjohnwu.magisk",
    "me.weishu.kernelsu",
];

/**
 * Bypass Emulator Detection
 * @param {any} function(
 * @returns {any}
 */
Java.perform(function() {

    Java.use("android.os.Build").PRODUCT.value = "gracerltexx";
    Java.use("android.os.Build").MANUFACTURER.value = "samsung";
    Java.use("android.os.Build").BRAND.value = "samsung";
    Java.use("android.os.Build").DEVICE.value = "gracerlte";
    Java.use("android.os.Build").MODEL.value = "SM-N935F";
    Java.use("android.os.Build").HARDWARE.value = "samsungexynos8890";
    Java.use("android.os.Build").FINGERPRINT.value =
        "samsung/gracerltexx/gracerlte:8.0.0/R16NW/N935FXXS4BRK2:user/release-keys";


    try {
        Java.use("java.io.File").exists.implementation = function() {
            var name = Java.use("java.io.File").getName.call(this);
            var catched = ["qemud", "qemu_pipe", "drivers", "cpuinfo"].indexOf(name) > -1;
            if (catched) {
                console.log("the pipe " + name + " existence is hooked");
                return false;
            } else {
                return this.exists.call(this);
            }
        };
    } catch (err) {
        console.log("[-] java.io.File.exists never called [-]");
    }

    // rename the package names
    try {
        Java.use("android.app.ApplicationPackageManager").getPackageInfo.overload(
            "java.lang.String",
            "int"
        ).implementation = function(name, flag) {
            var catched = ["com.example.android.apis", "com.android.development"].indexOf(name) >
                -1;
            if (catched) {
                console.log("the package " + name + " is renamed with fake name");
                name = "fake.package.name";
            }
            return this.getPackageInfo.call(this, name, flag);
        };
    } catch (err) {
        console.log(
            "[-] ApplicationPackageManager.getPackageInfo never called [-]"
        );
    }

    // hook the `android_getCpuFamily` method
    // https://android.googlesource.com/platform/ndk/+/master/sources/android/cpufeatures/cpu-features.c#1067
    // Note: If you pass "null" as the first parameter for "Module.findExportByName" it will search in all modules
    try {
        Interceptor.attach(Module.findExportByName(null, "android_getCpuFamily"), {
            onLeave: function(retval) {
                // const int ANDROID_CPU_FAMILY_X86 = 2;
                // const int ANDROID_CPU_FAMILY_X86_64 = 5;
                if ([2, 5].indexOf(retval) > -1) {
                    // const int ANDROID_CPU_FAMILY_ARM64 = 4;
                    retval.replace(4);
                }
            },
        });
    } catch (err) {
        console.log("[-] android_getCpuFamily never called [-]");
        // TODO: trace RegisterNatives in case the libraries are stripped.
    }
});

/**
 * Bypass Root Detection
 * @param {any} function(
 * @returns {any}
 */
setTimeout(function() {
    function stackTraceHere(isLog) {
        var Exception = Java.use("java.lang.Exception");
        var Log = Java.use("android.util.Log");
        var stackinfo = Log.getStackTraceString(Exception.$new());
        if (isLog) {
            console.log(stackinfo);
        } else {
            return stackinfo;
        }
    }

    function stackTraceNativeHere(isLog) {
        var backtrace = Thread.backtrace(this.context, Backtracer.ACCURATE)
            .map(DebugSymbol.fromAddress)
            .join("\n\t");
        console.log(backtrace);
    }

    function bypassJavaFileCheck() {
        var UnixFileSystem = Java.use("java.io.UnixFileSystem");
        UnixFileSystem.checkAccess.implementation = function(file, access) {
            var stack = stackTraceHere(false);

            const filename = file.getAbsolutePath();

            if (filename.indexOf("magisk") >= 0) {
                console.log("Anti Root Detect - check file: " + filename);
                return false;
            }

            if (commonPaths.indexOf(filename) >= 0) {
                console.log("Anti Root Detect - check file: " + filename);
                return false;
            }

            return this.checkAccess(file, access);
        };
    }

    function bypassNativeFileCheck() {
        var fopen = Module.findExportByName("libc.so", "fopen");
        Interceptor.attach(fopen, {
            onEnter: function(args) {
                this.inputPath = args[0].readUtf8String();
            },
            onLeave: function(retval) {
                if (retval.toInt32() != 0) {
                    if (commonPaths.indexOf(this.inputPath) >= 0) {
                        console.log("Anti Root Detect - fopen : " + this.inputPath);
                        retval.replace(ptr(0x0));
                    }
                }
            },
        });

        var access = Module.findExportByName("libc.so", "access");
        Interceptor.attach(access, {
            onEnter: function(args) {
                this.inputPath = args[0].readUtf8String();
            },
            onLeave: function(retval) {
                if (retval.toInt32() == 0) {
                    if (commonPaths.indexOf(this.inputPath) >= 0) {
                        console.log("Anti Root Detect - access : " + this.inputPath);
                        retval.replace(ptr(-1));
                    }
                }
            },
        });
    }

    function setProp() {
        var Build = Java.use("android.os.Build");
        var TAGS = Build.class.getDeclaredField("TAGS");
        TAGS.setAccessible(true);
        TAGS.set(null, "release-keys");

        var FINGERPRINT = Build.class.getDeclaredField("FINGERPRINT");
        FINGERPRINT.setAccessible(true);
        FINGERPRINT.set(
            null,
            "google/crosshatch/crosshatch:10/QQ3A.200805.001/6578210:user/release-keys"
        );

        // Build.deriveFingerprint.inplementation = function(){
        //     var ret = this.deriveFingerprint() //该函数无法通过反射调用
        //     console.log(ret)
        //     return ret
        // }

        var system_property_get = Module.findExportByName(
            "libc.so",
            "__system_property_get"
        );
        Interceptor.attach(system_property_get, {
            onEnter(args) {
                this.key = args[0].readCString();
                this.ret = args[1];
            },
            onLeave(ret) {
                if (this.key == "ro.build.fingerprint") {
                    var tmp =
                        "google/crosshatch/crosshatch:10/QQ3A.200805.001/6578210:user/release-keys";
                    var p = Memory.allocUtf8String(tmp);
                    Memory.copy(this.ret, p, tmp.length + 1);
                }
            },
        });
    }

    //android.app.PackageManager
    function bypassRootAppCheck() {
        var ApplicationPackageManager = Java.use(
            "android.app.ApplicationPackageManager"
        );
        ApplicationPackageManager.getPackageInfo.overload(
            "java.lang.String",
            "int"
        ).implementation = function(str, i) {
            // console.log(str)
            if (ROOTmanagementApp.indexOf(str) >= 0) {
                console.log("Anti Root Detect - check package : " + str);
                str = "ashen.one.ye.not.found";
            }
            return this.getPackageInfo(str, i);
        };

        //shell pm check
    }

    function bypassShellCheck() {
        var String = Java.use("java.lang.String");

        var ProcessImpl = Java.use("java.lang.ProcessImpl");
        ProcessImpl.start.implementation = function(
            cmdarray,
            env,
            dir,
            redirects,
            redirectErrorStream
        ) {
            if (cmdarray[0] == "mount") {
                console.log("Anti Root Detect - Shell : " + cmdarray.toString());
                arguments[0] = Java.array("java.lang.String", [String.$new("")]);
                return ProcessImpl.start.apply(this, arguments);
            }

            if (cmdarray[0] == "getprop") {
                console.log("Anti Root Detect - Shell : " + cmdarray.toString());
                const prop = ["ro.secure", "ro.debuggable"];
                if (prop.indexOf(cmdarray[1]) >= 0) {
                    arguments[0] = Java.array("java.lang.String", [String.$new("")]);
                    return ProcessImpl.start.apply(this, arguments);
                }
            }

            if (cmdarray[0].indexOf("which") >= 0) {
                const prop = ["su"];
                if (prop.indexOf(cmdarray[1]) >= 0) {
                    console.log("Anti Root Detect - Shell : " + cmdarray.toString());
                    arguments[0] = Java.array("java.lang.String", [String.$new("")]);
                    return ProcessImpl.start.apply(this, arguments);
                }
            }

            return ProcessImpl.start.apply(this, arguments);
        };
    }

    console.log("Attach");
    bypassNativeFileCheck();
    bypassJavaFileCheck();
    setProp();
    bypassRootAppCheck();
    bypassShellCheck();


    Java.perform(function() {
        var RootPackages = [
            "com.noshufou.android.su",
            "com.noshufou.android.su.elite",
            "eu.chainfire.supersu",
            "com.koushikdutta.superuser",
            "com.thirdparty.superuser",
            "com.yellowes.su",
            "com.koushikdutta.rommanager",
            "com.koushikdutta.rommanager.license",
            "com.dimonvideo.luckypatcher",
            "com.chelpus.lackypatch",
            "com.ramdroid.appquarantine",
            "com.ramdroid.appquarantinepro",
            "com.devadvance.rootcloak",
            "com.devadvance.rootcloakplus",
            "de.robv.android.xposed.installer",
            "com.saurik.substrate",
            "com.zachspong.temprootremovejb",
            "com.amphoras.hidemyroot",
            "com.amphoras.hidemyrootadfree",
            "com.formyhm.hiderootPremium",
            "com.formyhm.hideroot",
            "me.phh.superuser",
            "eu.chainfire.supersu.pro",
            "com.kingouser.com",
            "com.topjohnwu.magisk",
        ];

        var RootBinaries = [
            "su",
            "busybox",
            "supersu",
            "Superuser.apk",
            "KingoUser.apk",
            "SuperSu.apk",
            "magisk",
        ];

        var RootProperties = {
            "ro.build.selinux": "1",
            "ro.debuggable": "0",
            "service.adb.root": "0",
            "ro.secure": "1",
        };

        var RootPropertiesKeys = [];

        for (var k in RootProperties) RootPropertiesKeys.push(k);

        var PackageManager = Java.use("android.app.ApplicationPackageManager");

        var Runtime = Java.use("java.lang.Runtime");

        var NativeFile = Java.use("java.io.File");

        var String = Java.use("java.lang.String");

        var SystemProperties = Java.use("android.os.SystemProperties");

        var BufferedReader = Java.use("java.io.BufferedReader");

        var ProcessBuilder = Java.use("java.lang.ProcessBuilder");

        var StringBuffer = Java.use("java.lang.StringBuffer");

        var loaded_classes = Java.enumerateLoadedClassesSync();

        send("Loaded " + loaded_classes.length + " classes!");

        var useKeyInfo = false;

        var useProcessManager = false;

        send("loaded: " + loaded_classes.indexOf("java.lang.ProcessManager"));

        if (loaded_classes.indexOf("java.lang.ProcessManager") != -1) {
            try {
                //useProcessManager = true;
                //var ProcessManager = Java.use('java.lang.ProcessManager');
            } catch (err) {
                send("ProcessManager Hook failed: " + err);
            }
        } else {
            send("ProcessManager hook not loaded");
        }

        var KeyInfo = null;

        if (loaded_classes.indexOf("android.security.keystore.KeyInfo") != -1) {
            try {
                //useKeyInfo = true;
                //var KeyInfo = Java.use('android.security.keystore.KeyInfo');
            } catch (err) {
                send("KeyInfo Hook failed: " + err);
            }
        } else {
            send("KeyInfo hook not loaded");
        }

        PackageManager.getPackageInfo.overload(
            "java.lang.String",
            "int"
        ).implementation = function(pname, flags) {
            var shouldFakePackage = RootPackages.indexOf(pname) > -1;
            if (shouldFakePackage) {
                send("Bypass root check for package: " + pname);
                pname = "set.package.name.to.a.fake.one.so.we.can.bypass.it";
            }
            return this.getPackageInfo
                .overload("java.lang.String", "int")
                .call(this, pname, flags);
        };

        NativeFile.exists.implementation = function() {
            var name = NativeFile.getName.call(this);
            var shouldFakeReturn = RootBinaries.indexOf(name) > -1;
            if (shouldFakeReturn) {
                send("Bypass return value for binary: " + name);
                return false;
            } else {
                return this.exists.call(this);
            }
        };

        var exec = Runtime.exec.overload("[Ljava.lang.String;");
        var exec1 = Runtime.exec.overload("java.lang.String");
        var exec2 = Runtime.exec.overload("java.lang.String", "[Ljava.lang.String;");
        var exec3 = Runtime.exec.overload(
            "[Ljava.lang.String;",
            "[Ljava.lang.String;"
        );
        var exec4 = Runtime.exec.overload(
            "[Ljava.lang.String;",
            "[Ljava.lang.String;",
            "java.io.File"
        );
        var exec5 = Runtime.exec.overload(
            "java.lang.String",
            "[Ljava.lang.String;",
            "java.io.File"
        );

        exec5.implementation = function(cmd, env, dir) {
            if (
                cmd.indexOf("getprop") != -1 ||
                cmd == "mount" ||
                cmd.indexOf("build.prop") != -1 ||
                cmd == "id" ||
                cmd == "sh"
            ) {
                var fakeCmd = "grep";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            if (cmd == "su") {
                var fakeCmd =
                    "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            return exec5.call(this, cmd, env, dir);
        };

        exec4.implementation = function(cmdarr, env, file) {
            for (var i = 0; i < cmdarr.length; i = i + 1) {
                var tmp_cmd = cmdarr[i];
                if (
                    tmp_cmd.indexOf("getprop") != -1 ||
                    tmp_cmd == "mount" ||
                    tmp_cmd.indexOf("build.prop") != -1 ||
                    tmp_cmd == "id" ||
                    tmp_cmd == "sh"
                ) {
                    var fakeCmd = "grep";
                    send("Bypass " + cmdarr + " command");
                    return exec1.call(this, fakeCmd);
                }

                if (tmp_cmd == "su") {
                    var fakeCmd =
                        "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                    send("Bypass " + cmdarr + " command");
                    return exec1.call(this, fakeCmd);
                }
            }
            return exec4.call(this, cmdarr, env, file);
        };

        exec3.implementation = function(cmdarr, envp) {
            for (var i = 0; i < cmdarr.length; i = i + 1) {
                var tmp_cmd = cmdarr[i];
                if (
                    tmp_cmd.indexOf("getprop") != -1 ||
                    tmp_cmd == "mount" ||
                    tmp_cmd.indexOf("build.prop") != -1 ||
                    tmp_cmd == "id" ||
                    tmp_cmd == "sh"
                ) {
                    var fakeCmd = "grep";
                    send("Bypass " + cmdarr + " command");
                    return exec1.call(this, fakeCmd);
                }

                if (tmp_cmd == "su") {
                    var fakeCmd =
                        "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                    send("Bypass " + cmdarr + " command");
                    return exec1.call(this, fakeCmd);
                }
            }
            return exec3.call(this, cmdarr, envp);
        };

        exec2.implementation = function(cmd, env) {
            if (
                cmd.indexOf("getprop") != -1 ||
                cmd == "mount" ||
                cmd.indexOf("build.prop") != -1 ||
                cmd == "id" ||
                cmd == "sh"
            ) {
                var fakeCmd = "grep";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            if (cmd == "su") {
                var fakeCmd =
                    "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            return exec2.call(this, cmd, env);
        };

        exec.implementation = function(cmd) {
            for (var i = 0; i < cmd.length; i = i + 1) {
                var tmp_cmd = cmd[i];
                if (
                    tmp_cmd.indexOf("getprop") != -1 ||
                    tmp_cmd == "mount" ||
                    tmp_cmd.indexOf("build.prop") != -1 ||
                    tmp_cmd == "id" ||
                    tmp_cmd == "sh"
                ) {
                    var fakeCmd = "grep";
                    send("Bypass " + cmd + " command");
                    return exec1.call(this, fakeCmd);
                }

                if (tmp_cmd == "su") {
                    var fakeCmd =
                        "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                    send("Bypass " + cmd + " command");
                    return exec1.call(this, fakeCmd);
                }
            }

            return exec.call(this, cmd);
        };

        exec1.implementation = function(cmd) {
            if (
                cmd.indexOf("getprop") != -1 ||
                cmd == "mount" ||
                cmd.indexOf("build.prop") != -1 ||
                cmd == "id" ||
                cmd == "sh"
            ) {
                var fakeCmd = "grep";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            if (cmd == "su") {
                var fakeCmd =
                    "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled";
                send("Bypass " + cmd + " command");
                return exec1.call(this, fakeCmd);
            }
            return exec1.call(this, cmd);
        };

        String.contains.implementation = function(name) {
            if (name == "test-keys") {
                send("Bypass test-keys check");
                return false;
            }
            return this.contains.call(this, name);
        };

        var get = SystemProperties.get.overload("java.lang.String");

        get.implementation = function(name) {
            if (RootPropertiesKeys.indexOf(name) != -1) {
                send("Bypass " + name);
                return RootProperties[name];
            }
            return this.get.call(this, name);
        };

        Interceptor.attach(Module.findExportByName("libc.so", "fopen"), {
            onEnter: function(args) {
                var path = Memory.readCString(args[0]);
                path = path.split("/");
                var executable = path[path.length - 1];
                var shouldFakeReturn = RootBinaries.indexOf(executable) > -1;
                if (shouldFakeReturn) {
                    Memory.writeUtf8String(args[0], "/notexists");
                    send("Bypass native fopen");
                }
            },
            onLeave: function(retval) {},
        });

        Interceptor.attach(Module.findExportByName("libc.so", "system"), {
            onEnter: function(args) {
                var cmd = Memory.readCString(args[0]);
                send("SYSTEM CMD: " + cmd);
                if (
                    cmd.indexOf("getprop") != -1 ||
                    cmd == "mount" ||
                    cmd.indexOf("build.prop") != -1 ||
                    cmd == "id"
                ) {
                    send("Bypass native system: " + cmd);
                    Memory.writeUtf8String(args[0], "grep");
                }
                if (cmd == "su") {
                    send("Bypass native system: " + cmd);
                    Memory.writeUtf8String(
                        args[0],
                        "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled"
                    );
                }
            },
            onLeave: function(retval) {},
        });

        /*

        TO IMPLEMENT:

        Exec Family

        int execl(const char *path, const char *arg0, ..., const char *argn, (char *)0);
        int execle(const char *path, const char *arg0, ..., const char *argn, (char *)0, char *const envp[]);
        int execlp(const char *file, const char *arg0, ..., const char *argn, (char *)0);
        int execlpe(const char *file, const char *arg0, ..., const char *argn, (char *)0, char *const envp[]);
        int execv(const char *path, char *const argv[]);
        int execve(const char *path, char *const argv[], char *const envp[]);
        int execvp(const char *file, char *const argv[]);
        int execvpe(const char *file, char *const argv[], char *const envp[]);

        */

        BufferedReader.readLine.overload("boolean").implementation = function() {
            var text = this.readLine.overload("boolean").call(this);
            if (text === null) {
                // just pass , i know it's ugly as hell but test != null won't work :(
            } else {
                var shouldFakeRead = text.indexOf("ro.build.tags=test-keys") > -1;
                if (shouldFakeRead) {
                    send("Bypass build.prop file read");
                    text = text.replace(
                        "ro.build.tags=test-keys",
                        "ro.build.tags=release-keys"
                    );
                }
            }
            return text;
        };

        var executeCommand = ProcessBuilder.command.overload("java.util.List");

        ProcessBuilder.start.implementation = function() {
            var cmd = this.command.call(this);
            var shouldModifyCommand = false;
            for (var i = 0; i < cmd.size(); i = i + 1) {
                var tmp_cmd = cmd.get(i).toString();
                if (
                    tmp_cmd.indexOf("getprop") != -1 ||
                    tmp_cmd.indexOf("mount") != -1 ||
                    tmp_cmd.indexOf("build.prop") != -1 ||
                    tmp_cmd.indexOf("id") != -1
                ) {
                    shouldModifyCommand = true;
                }
            }
            if (shouldModifyCommand) {
                send("Bypass ProcessBuilder " + cmd);
                this.command.call(this, ["grep"]);
                return this.start.call(this);
            }
            if (cmd.indexOf("su") != -1) {
                send("Bypass ProcessBuilder " + cmd);
                this.command.call(this, [
                    "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled",
                ]);
                return this.start.call(this);
            }

            return this.start.call(this);
        };

        if (useProcessManager) {
            var ProcManExec = ProcessManager.exec.overload(
                "[Ljava.lang.String;",
                "[Ljava.lang.String;",
                "java.io.File",
                "boolean"
            );
            var ProcManExecVariant = ProcessManager.exec.overload(
                "[Ljava.lang.String;",
                "[Ljava.lang.String;",
                "java.lang.String",
                "java.io.FileDescriptor",
                "java.io.FileDescriptor",
                "java.io.FileDescriptor",
                "boolean"
            );

            ProcManExec.implementation = function(cmd, env, workdir, redirectstderr) {
                var fake_cmd = cmd;
                for (var i = 0; i < cmd.length; i = i + 1) {
                    var tmp_cmd = cmd[i];
                    if (
                        tmp_cmd.indexOf("getprop") != -1 ||
                        tmp_cmd == "mount" ||
                        tmp_cmd.indexOf("build.prop") != -1 ||
                        tmp_cmd == "id"
                    ) {
                        var fake_cmd = ["grep"];
                        send("Bypass " + cmdarr + " command");
                    }

                    if (tmp_cmd == "su") {
                        var fake_cmd = [
                            "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled",
                        ];
                        send("Bypass " + cmdarr + " command");
                    }
                }
                return ProcManExec.call(this, fake_cmd, env, workdir, redirectstderr);
            };

            ProcManExecVariant.implementation = function(
                cmd,
                env,
                directory,
                stdin,
                stdout,
                stderr,
                redirect
            ) {
                var fake_cmd = cmd;
                for (var i = 0; i < cmd.length; i = i + 1) {
                    var tmp_cmd = cmd[i];
                    if (
                        tmp_cmd.indexOf("getprop") != -1 ||
                        tmp_cmd == "mount" ||
                        tmp_cmd.indexOf("build.prop") != -1 ||
                        tmp_cmd == "id"
                    ) {
                        var fake_cmd = ["grep"];
                        send("Bypass " + cmdarr + " command");
                    }

                    if (tmp_cmd == "su") {
                        var fake_cmd = [
                            "justafakecommandthatcannotexistsusingthisshouldthowanexceptionwheneversuiscalled",
                        ];
                        send("Bypass " + cmdarr + " command");
                    }
                }
                return ProcManExecVariant.call(
                    this,
                    fake_cmd,
                    env,
                    directory,
                    stdin,
                    stdout,
                    stderr,
                    redirect
                );
            };
        }

        if (useKeyInfo) {
            KeyInfo.isInsideSecureHardware.implementation = function() {
                send("Bypass isInsideSecureHardware");
                return true;
            };
        }
    });

}, 0);

/**
 * Bypass Multiple SSL Pinning
 * @param {any} function(
 * @returns {any}
 */
setTimeout(function() {
    Java.perform(function() {
        console.log("---");
        console.log("Unpinning Android app...");

        /// -- Generic hook to protect against SSLPeerUnverifiedException -- ///

        // In some cases, with unusual cert pinning approaches, or heavy obfuscation, we can't
        // match the real method & package names. This is a problem! Fortunately, we can still
        // always match built-in types, so here we spot all failures that use the built-in cert
        // error type (notably this includes OkHttp), and after the first failure, we dynamically
        // generate & inject a patch to completely disable the method that threw the error.
        try {
            const UnverifiedCertError = Java.use(
                "javax.net.ssl.SSLPeerUnverifiedException"
            );
            UnverifiedCertError.$init.implementation = function(str) {
                console.log(
                    "  --> Unexpected SSL verification failure, adding dynamic patch..."
                );

                try {
                    const stackTrace = Java.use("java.lang.Thread")
                        .currentThread()
                        .getStackTrace();
                    const exceptionStackIndex = stackTrace.findIndex(
                        (stack) =>
                        stack.getClassName() ===
                        "javax.net.ssl.SSLPeerUnverifiedException"
                    );
                    const callingFunctionStack = stackTrace[exceptionStackIndex + 1];

                    const className = callingFunctionStack.getClassName();
                    const methodName = callingFunctionStack.getMethodName();

                    console.log(`      Thrown by ${className}->${methodName}`);

                    const callingClass = Java.use(className);
                    const callingMethod = callingClass[methodName];

                    if (callingMethod.implementation) return; // Already patched by Frida - skip it

                    console.log("      Attempting to patch automatically...");
                    const returnTypeName = callingMethod.returnType.type;

                    callingMethod.implementation = function() {
                        console.log(
                            `  --> Bypassing ${className}->${methodName} (automatic exception patch)`
                        );

                        // This is not a perfect fix! Most unknown cases like this are really just
                        // checkCert(cert) methods though, so doing nothing is perfect, and if we
                        // do need an actual return value then this is probably the best we can do,
                        // and at least we're logging the method name so you can patch it manually:

                        if (returnTypeName === "void") {
                            return;
                        } else {
                            return null;
                        }
                    };

                    console.log(
                        `      [+] ${className}->${methodName} (automatic exception patch)`
                    );
                } catch (e) {
                    console.log("      [ ] Failed to automatically patch failure");
                }

                return this.$init(str);
            };
            console.log("[+] SSLPeerUnverifiedException auto-patcher");
        } catch (err) {
            console.log("[ ] SSLPeerUnverifiedException auto-patcher");
        }

        /// -- Specific targeted hooks: -- ///

        // HttpsURLConnection
        try {
            const HttpsURLConnection = Java.use("javax.net.ssl.HttpsURLConnection");
            HttpsURLConnection.setDefaultHostnameVerifier.implementation = function(
                hostnameVerifier
            ) {
                console.log(
                    "  --> Bypassing HttpsURLConnection (setDefaultHostnameVerifier)"
                );
                return; // Do nothing, i.e. don't change the hostname verifier
            };
            console.log("[+] HttpsURLConnection (setDefaultHostnameVerifier)");
        } catch (err) {
            console.log("[ ] HttpsURLConnection (setDefaultHostnameVerifier)");
        }
        try {
            const HttpsURLConnection = Java.use("javax.net.ssl.HttpsURLConnection");
            HttpsURLConnection.setSSLSocketFactory.implementation = function(
                SSLSocketFactory
            ) {
                console.log("  --> Bypassing HttpsURLConnection (setSSLSocketFactory)");
                return; // Do nothing, i.e. don't change the SSL socket factory
            };
            console.log("[+] HttpsURLConnection (setSSLSocketFactory)");
        } catch (err) {
            console.log("[ ] HttpsURLConnection (setSSLSocketFactory)");
        }
        try {
            const HttpsURLConnection = Java.use("javax.net.ssl.HttpsURLConnection");
            HttpsURLConnection.setHostnameVerifier.implementation = function(
                hostnameVerifier
            ) {
                console.log("  --> Bypassing HttpsURLConnection (setHostnameVerifier)");
                return; // Do nothing, i.e. don't change the hostname verifier
            };
            console.log("[+] HttpsURLConnection (setHostnameVerifier)");
        } catch (err) {
            console.log("[ ] HttpsURLConnection (setHostnameVerifier)");
        }

        // SSLContext
        try {
            const X509TrustManager = Java.use("javax.net.ssl.X509TrustManager");
            const SSLContext = Java.use("javax.net.ssl.SSLContext");

            const TrustManager = Java.registerClass({
                // Implement a custom TrustManager
                name: "dev.asd.test.TrustManager",
                implements: [X509TrustManager],
                methods: {
                    checkClientTrusted: function(chain, authType) {},
                    checkServerTrusted: function(chain, authType) {},
                    getAcceptedIssuers: function() {
                        return [];
                    },
                },
            });

            // Prepare the TrustManager array to pass to SSLContext.init()
            const TrustManagers = [TrustManager.$new()];

            // Get a handle on the init() on the SSLContext class
            const SSLContext_init = SSLContext.init.overload(
                "[Ljavax.net.ssl.KeyManager;",
                "[Ljavax.net.ssl.TrustManager;",
                "java.security.SecureRandom"
            );

            // Override the init method, specifying the custom TrustManager
            SSLContext_init.implementation = function(
                keyManager,
                trustManager,
                secureRandom
            ) {
                console.log("  --> Bypassing Trustmanager (Android < 7) request");
                SSLContext_init.call(this, keyManager, TrustManagers, secureRandom);
            };
            console.log("[+] SSLContext");
        } catch (err) {
            console.log("[ ] SSLContext");
        }

        // TrustManagerImpl (Android > 7)
        try {
            const array_list = Java.use("java.util.ArrayList");
            const TrustManagerImpl = Java.use(
                "com.android.org.conscrypt.TrustManagerImpl"
            );

            // This step is notably what defeats the most common case: network security config
            TrustManagerImpl.checkTrustedRecursive.implementation = function(
                a1,
                a2,
                a3,
                a4,
                a5,
                a6
            ) {
                console.log("  --> Bypassing TrustManagerImpl checkTrusted ");
                return array_list.$new();
            };

            TrustManagerImpl.verifyChain.implementation = function(
                untrustedChain,
                trustAnchorChain,
                host,
                clientAuth,
                ocspData,
                tlsSctData
            ) {
                console.log("  --> Bypassing TrustManagerImpl verifyChain: " + host);
                return untrustedChain;
            };
            console.log("[+] TrustManagerImpl");
        } catch (err) {
            console.log("[ ] TrustManagerImpl");
        }

        // OkHTTPv3 (quadruple bypass)
        try {
            // Bypass OkHTTPv3 {1}
            const okhttp3_Activity_1 = Java.use("okhttp3.CertificatePinner");
            okhttp3_Activity_1.check.overload(
                "java.lang.String",
                "java.util.List"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing OkHTTPv3 (list): " + a);
                return;
            };
            console.log("[+] OkHTTPv3 (list)");
        } catch (err) {
            console.log("[ ] OkHTTPv3 (list)");
        }
        try {
            // Bypass OkHTTPv3 {2}
            // This method of CertificatePinner.check could be found in some old Android app
            const okhttp3_Activity_2 = Java.use("okhttp3.CertificatePinner");
            okhttp3_Activity_2.check.overload(
                "java.lang.String",
                "java.security.cert.Certificate"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing OkHTTPv3 (cert): " + a);
                return;
            };
            console.log("[+] OkHTTPv3 (cert)");
        } catch (err) {
            console.log("[ ] OkHTTPv3 (cert)");
        }
        try {
            // Bypass OkHTTPv3 {3}
            const okhttp3_Activity_3 = Java.use("okhttp3.CertificatePinner");
            okhttp3_Activity_3.check.overload(
                "java.lang.String",
                "[Ljava.security.cert.Certificate;"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing OkHTTPv3 (cert array): " + a);
                return;
            };
            console.log("[+] OkHTTPv3 (cert array)");
        } catch (err) {
            console.log("[ ] OkHTTPv3 (cert array)");
        }
        try {
            // Bypass OkHTTPv3 {4}
            const okhttp3_Activity_4 = Java.use("okhttp3.CertificatePinner");
            okhttp3_Activity_4["check$okhttp"].implementation = function(a, b) {
                console.log("  --> Bypassing OkHTTPv3 ($okhttp): " + a);
                return;
            };
            console.log("[+] OkHTTPv3 ($okhttp)");
        } catch (err) {
            console.log("[ ] OkHTTPv3 ($okhttp)");
        }

        // Trustkit (triple bypass)
        try {
            // Bypass Trustkit {1}
            const trustkit_Activity_1 = Java.use(
                "com.datatheorem.android.trustkit.pinning.OkHostnameVerifier"
            );
            trustkit_Activity_1.verify.overload(
                "java.lang.String",
                "javax.net.ssl.SSLSession"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing Trustkit OkHostnameVerifier(SSLSession): " + a
                );
                return true;
            };
            console.log("[+] Trustkit OkHostnameVerifier(SSLSession)");
        } catch (err) {
            console.log("[ ] Trustkit OkHostnameVerifier(SSLSession)");
        }
        try {
            // Bypass Trustkit {2}
            const trustkit_Activity_2 = Java.use(
                "com.datatheorem.android.trustkit.pinning.OkHostnameVerifier"
            );
            trustkit_Activity_2.verify.overload(
                "java.lang.String",
                "java.security.cert.X509Certificate"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing Trustkit OkHostnameVerifier(cert): " + a);
                return true;
            };
            console.log("[+] Trustkit OkHostnameVerifier(cert)");
        } catch (err) {
            console.log("[ ] Trustkit OkHostnameVerifier(cert)");
        }
        try {
            // Bypass Trustkit {3}
            const trustkit_PinningTrustManager = Java.use(
                "com.datatheorem.android.trustkit.pinning.PinningTrustManager"
            );
            trustkit_PinningTrustManager.checkServerTrusted.implementation =
                function() {
                    console.log("  --> Bypassing Trustkit PinningTrustManager");
                };
            console.log("[+] Trustkit PinningTrustManager");
        } catch (err) {
            console.log("[ ] Trustkit PinningTrustManager");
        }

        // Appcelerator Titanium
        try {
            const appcelerator_PinningTrustManager = Java.use(
                "appcelerator.https.PinningTrustManager"
            );
            appcelerator_PinningTrustManager.checkServerTrusted.implementation =
                function() {
                    console.log("  --> Bypassing Appcelerator PinningTrustManager");
                };
            console.log("[+] Appcelerator PinningTrustManager");
        } catch (err) {
            console.log("[ ] Appcelerator PinningTrustManager");
        }

        // OpenSSLSocketImpl Conscrypt
        try {
            const OpenSSLSocketImpl = Java.use(
                "com.android.org.conscrypt.OpenSSLSocketImpl"
            );
            OpenSSLSocketImpl.verifyCertificateChain.implementation = function(
                certRefs,
                JavaObject,
                authMethod
            ) {
                console.log("  --> Bypassing OpenSSLSocketImpl Conscrypt");
            };
            console.log("[+] OpenSSLSocketImpl Conscrypt");
        } catch (err) {
            console.log("[ ] OpenSSLSocketImpl Conscrypt");
        }

        // OpenSSLEngineSocketImpl Conscrypt
        try {
            const OpenSSLEngineSocketImpl_Activity = Java.use(
                "com.android.org.conscrypt.OpenSSLEngineSocketImpl"
            );
            OpenSSLEngineSocketImpl_Activity.verifyCertificateChain.overload(
                "[Ljava.lang.Long;",
                "java.lang.String"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing OpenSSLEngineSocketImpl Conscrypt: " + b);
            };
            console.log("[+] OpenSSLEngineSocketImpl Conscrypt");
        } catch (err) {
            console.log("[ ] OpenSSLEngineSocketImpl Conscrypt");
        }

        // OpenSSLSocketImpl Apache Harmony
        try {
            const OpenSSLSocketImpl_Harmony = Java.use(
                "org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl"
            );
            OpenSSLSocketImpl_Harmony.verifyCertificateChain.implementation =
                function(asn1DerEncodedCertificateChain, authMethod) {
                    console.log("  --> Bypassing OpenSSLSocketImpl Apache Harmony");
                };
            console.log("[+] OpenSSLSocketImpl Apache Harmony");
        } catch (err) {
            console.log("[ ] OpenSSLSocketImpl Apache Harmony");
        }

        // PhoneGap sslCertificateChecker (https://github.com/EddyVerbruggen/SSLCertificateChecker-PhoneGap-Plugin)
        try {
            const phonegap_Activity = Java.use(
                "nl.xservices.plugins.sslCertificateChecker"
            );
            phonegap_Activity.execute.overload(
                "java.lang.String",
                "org.json.JSONArray",
                "org.apache.cordova.CallbackContext"
            ).implementation = function(a, b, c) {
                console.log("  --> Bypassing PhoneGap sslCertificateChecker: " + a);
                return true;
            };
            console.log("[+] PhoneGap sslCertificateChecker");
        } catch (err) {
            console.log("[ ] PhoneGap sslCertificateChecker");
        }

        // IBM MobileFirst pinTrustedCertificatePublicKey (double bypass)
        try {
            // Bypass IBM MobileFirst {1}
            const WLClient_Activity_1 = Java.use(
                "com.worklight.wlclient.api.WLClient"
            );
            WLClient_Activity_1.getInstance().pinTrustedCertificatePublicKey.overload(
                "java.lang.String"
            ).implementation = function(cert) {
                console.log(
                    "  --> Bypassing IBM MobileFirst pinTrustedCertificatePublicKey (string): " +
                    cert
                );
                return;
            };
            console.log(
                "[+] IBM MobileFirst pinTrustedCertificatePublicKey (string)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM MobileFirst pinTrustedCertificatePublicKey (string)"
            );
        }
        try {
            // Bypass IBM MobileFirst {2}
            const WLClient_Activity_2 = Java.use(
                "com.worklight.wlclient.api.WLClient"
            );
            WLClient_Activity_2.getInstance().pinTrustedCertificatePublicKey.overload(
                "[Ljava.lang.String;"
            ).implementation = function(cert) {
                console.log(
                    "  --> Bypassing IBM MobileFirst pinTrustedCertificatePublicKey (string array): " +
                    cert
                );
                return;
            };
            console.log(
                "[+] IBM MobileFirst pinTrustedCertificatePublicKey (string array)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM MobileFirst pinTrustedCertificatePublicKey (string array)"
            );
        }

        // IBM WorkLight (ancestor of MobileFirst) HostNameVerifierWithCertificatePinning (quadruple bypass)
        try {
            // Bypass IBM WorkLight {1}
            const worklight_Activity_1 = Java.use(
                "com.worklight.wlclient.certificatepinning.HostNameVerifierWithCertificatePinning"
            );
            worklight_Activity_1.verify.overload(
                "java.lang.String",
                "javax.net.ssl.SSLSocket"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSocket): " +
                    a
                );
                return;
            };
            console.log(
                "[+] IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSocket)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSocket)"
            );
        }
        try {
            // Bypass IBM WorkLight {2}
            const worklight_Activity_2 = Java.use(
                "com.worklight.wlclient.certificatepinning.HostNameVerifierWithCertificatePinning"
            );
            worklight_Activity_2.verify.overload(
                "java.lang.String",
                "java.security.cert.X509Certificate"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing IBM WorkLight HostNameVerifierWithCertificatePinning (cert): " +
                    a
                );
                return;
            };
            console.log(
                "[+] IBM WorkLight HostNameVerifierWithCertificatePinning (cert)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM WorkLight HostNameVerifierWithCertificatePinning (cert)"
            );
        }
        try {
            // Bypass IBM WorkLight {3}
            const worklight_Activity_3 = Java.use(
                "com.worklight.wlclient.certificatepinning.HostNameVerifierWithCertificatePinning"
            );
            worklight_Activity_3.verify.overload(
                "java.lang.String",
                "[Ljava.lang.String;",
                "[Ljava.lang.String;"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing IBM WorkLight HostNameVerifierWithCertificatePinning (string string): " +
                    a
                );
                return;
            };
            console.log(
                "[+] IBM WorkLight HostNameVerifierWithCertificatePinning (string string)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM WorkLight HostNameVerifierWithCertificatePinning (string string)"
            );
        }
        try {
            // Bypass IBM WorkLight {4}
            const worklight_Activity_4 = Java.use(
                "com.worklight.wlclient.certificatepinning.HostNameVerifierWithCertificatePinning"
            );
            worklight_Activity_4.verify.overload(
                "java.lang.String",
                "javax.net.ssl.SSLSession"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSession): " +
                    a
                );
                return true;
            };
            console.log(
                "[+] IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSession)"
            );
        } catch (err) {
            console.log(
                "[ ] IBM WorkLight HostNameVerifierWithCertificatePinning (SSLSession)"
            );
        }

        // Conscrypt CertPinManager
        try {
            const conscrypt_CertPinManager_Activity = Java.use(
                "com.android.org.conscrypt.CertPinManager"
            );
            conscrypt_CertPinManager_Activity.isChainValid.overload(
                "java.lang.String",
                "java.util.List"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing Conscrypt CertPinManager: " + a);
                return true;
            };
            console.log("[+] Conscrypt CertPinManager");
        } catch (err) {
            console.log("[ ] Conscrypt CertPinManager");
        }

        // CWAC-Netsecurity (unofficial back-port pinner for Android<4.2) CertPinManager
        try {
            const cwac_CertPinManager_Activity = Java.use(
                "com.commonsware.cwac.netsecurity.conscrypt.CertPinManager"
            );
            cwac_CertPinManager_Activity.isChainValid.overload(
                "java.lang.String",
                "java.util.List"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing CWAC-Netsecurity CertPinManager: " + a);
                return true;
            };
            console.log("[+] CWAC-Netsecurity CertPinManager");
        } catch (err) {
            console.log("[ ] CWAC-Netsecurity CertPinManager");
        }

        // Worklight Androidgap WLCertificatePinningPlugin
        try {
            const androidgap_WLCertificatePinningPlugin_Activity = Java.use(
                "com.worklight.androidgap.plugin.WLCertificatePinningPlugin"
            );
            androidgap_WLCertificatePinningPlugin_Activity.execute.overload(
                "java.lang.String",
                "org.json.JSONArray",
                "org.apache.cordova.CallbackContext"
            ).implementation = function(a, b, c) {
                console.log(
                    "  --> Bypassing Worklight Androidgap WLCertificatePinningPlugin: " +
                    a
                );
                return true;
            };
            console.log("[+] Worklight Androidgap WLCertificatePinningPlugin");
        } catch (err) {
            console.log("[ ] Worklight Androidgap WLCertificatePinningPlugin");
        }

        // Netty FingerprintTrustManagerFactory
        try {
            const netty_FingerprintTrustManagerFactory = Java.use(
                "io.netty.handler.ssl.util.FingerprintTrustManagerFactory"
            );
            netty_FingerprintTrustManagerFactory.checkTrusted.implementation =
                function(type, chain) {
                    console.log("  --> Bypassing Netty FingerprintTrustManagerFactory");
                };
            console.log("[+] Netty FingerprintTrustManagerFactory");
        } catch (err) {
            console.log("[ ] Netty FingerprintTrustManagerFactory");
        }

        // Squareup CertificatePinner [OkHTTP<v3] (double bypass)
        try {
            // Bypass Squareup CertificatePinner {1}
            const Squareup_CertificatePinner_Activity_1 = Java.use(
                "com.squareup.okhttp.CertificatePinner"
            );
            Squareup_CertificatePinner_Activity_1.check.overload(
                "java.lang.String",
                "java.security.cert.Certificate"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing Squareup CertificatePinner (cert): " + a);
                return;
            };
            console.log("[+] Squareup CertificatePinner (cert)");
        } catch (err) {
            console.log("[ ] Squareup CertificatePinner (cert)");
        }
        try {
            // Bypass Squareup CertificatePinner {2}
            const Squareup_CertificatePinner_Activity_2 = Java.use(
                "com.squareup.okhttp.CertificatePinner"
            );
            Squareup_CertificatePinner_Activity_2.check.overload(
                "java.lang.String",
                "java.util.List"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing Squareup CertificatePinner (list): " + a);
                return;
            };
            console.log("[+] Squareup CertificatePinner (list)");
        } catch (err) {
            console.log("[ ] Squareup CertificatePinner (list)");
        }

        // Squareup OkHostnameVerifier [OkHTTP v3] (double bypass)
        try {
            // Bypass Squareup OkHostnameVerifier {1}
            const Squareup_OkHostnameVerifier_Activity_1 = Java.use(
                "com.squareup.okhttp.internal.tls.OkHostnameVerifier"
            );
            Squareup_OkHostnameVerifier_Activity_1.verify.overload(
                "java.lang.String",
                "java.security.cert.X509Certificate"
            ).implementation = function(a, b) {
                console.log("  --> Bypassing Squareup OkHostnameVerifier (cert): " + a);
                return true;
            };
            console.log("[+] Squareup OkHostnameVerifier (cert)");
        } catch (err) {
            console.log("[ ] Squareup OkHostnameVerifier (cert)");
        }
        try {
            // Bypass Squareup OkHostnameVerifier {2}
            const Squareup_OkHostnameVerifier_Activity_2 = Java.use(
                "com.squareup.okhttp.internal.tls.OkHostnameVerifier"
            );
            Squareup_OkHostnameVerifier_Activity_2.verify.overload(
                "java.lang.String",
                "javax.net.ssl.SSLSession"
            ).implementation = function(a, b) {
                console.log(
                    "  --> Bypassing Squareup OkHostnameVerifier (SSLSession): " + a
                );
                return true;
            };
            console.log("[+] Squareup OkHostnameVerifier (SSLSession)");
        } catch (err) {
            console.log("[ ] Squareup OkHostnameVerifier (SSLSession)");
        }

        // Android WebViewClient (double bypass)
        try {
            // Bypass WebViewClient {1} (deprecated from Android 6)
            const AndroidWebViewClient_Activity_1 = Java.use(
                "android.webkit.WebViewClient"
            );
            AndroidWebViewClient_Activity_1.onReceivedSslError.overload(
                "android.webkit.WebView",
                "android.webkit.SslErrorHandler",
                "android.net.http.SslError"
            ).implementation = function(obj1, obj2, obj3) {
                console.log("  --> Bypassing Android WebViewClient (SslErrorHandler)");
            };
            console.log("[+] Android WebViewClient (SslErrorHandler)");
        } catch (err) {
            console.log("[ ] Android WebViewClient (SslErrorHandler)");
        }
        try {
            // Bypass WebViewClient {2}
            const AndroidWebViewClient_Activity_2 = Java.use(
                "android.webkit.WebViewClient"
            );
            AndroidWebViewClient_Activity_2.onReceivedSslError.overload(
                "android.webkit.WebView",
                "android.webkit.WebResourceRequest",
                "android.webkit.WebResourceError"
            ).implementation = function(obj1, obj2, obj3) {
                console.log("  --> Bypassing Android WebViewClient (WebResourceError)");
            };
            console.log("[+] Android WebViewClient (WebResourceError)");
        } catch (err) {
            console.log("[ ] Android WebViewClient (WebResourceError)");
        }

        // Apache Cordova WebViewClient
        try {
            const CordovaWebViewClient_Activity = Java.use(
                "org.apache.cordova.CordovaWebViewClient"
            );
            CordovaWebViewClient_Activity.onReceivedSslError.overload(
                "android.webkit.WebView",
                "android.webkit.SslErrorHandler",
                "android.net.http.SslError"
            ).implementation = function(obj1, obj2, obj3) {
                console.log("  --> Bypassing Apache Cordova WebViewClient");
                obj3.proceed();
            };
        } catch (err) {
            console.log("[ ] Apache Cordova WebViewClient");
        }

        // Boye AbstractVerifier
        try {
            const boye_AbstractVerifier = Java.use(
                "ch.boye.httpclientandroidlib.conn.ssl.AbstractVerifier"
            );
            boye_AbstractVerifier.verify.implementation = function(host, ssl) {
                console.log("  --> Bypassing Boye AbstractVerifier: " + host);
            };
        } catch (err) {
            console.log("[ ] Boye AbstractVerifier");
        }

        // Appmattus
        try {
            const appmatus_Activity = Java.use(
                "com.appmattus.certificatetransparency.internal.verifier.CertificateTransparencyInterceptor"
            );
            appmatus_Activity["intercept"].implementation = function(a) {
                console.log("  --> Bypassing Appmattus (Transparency)");
                return a.proceed(a.request());
            };
            console.log("[+] Appmattus (CertificateTransparencyInterceptor)");
        } catch (err) {
            console.log("[ ] Appmattus (CertificateTransparencyInterceptor)");
        }

        try {
            const CertificateTransparencyTrustManager = Java.use(
                "com.appmattus.certificatetransparency.internal.verifier.CertificateTransparencyTrustManager"
            );
            CertificateTransparencyTrustManager["checkServerTrusted"].overload(
                "[Ljava.security.cert.X509Certificate;",
                "java.lang.String"
            ).implementation = function(x509CertificateArr, str) {
                console.log(
                    "  --> Bypassing Appmattus (CertificateTransparencyTrustManager)"
                );
            };
            CertificateTransparencyTrustManager["checkServerTrusted"].overload(
                "[Ljava.security.cert.X509Certificate;",
                "java.lang.String",
                "java.lang.String"
            ).implementation = function(x509CertificateArr, str, str2) {
                console.log(
                    "  --> Bypassing Appmattus (CertificateTransparencyTrustManager)"
                );
                return Java.use("java.util.ArrayList").$new();
            };
            console.log("[+] Appmattus (CertificateTransparencyTrustManager)");
        } catch (err) {
            console.log("[ ] Appmattus (CertificateTransparencyTrustManager)");
        }

        console.log("Unpinning setup completed");
        console.log("---");
    });
}, 0);
```


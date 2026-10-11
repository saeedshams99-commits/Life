name: Build APK

on:
  push:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: Signing key
        run: |
          if [ ! -f life.keystore ]; then
            keytool -genkeypair -keystore life.keystore -storetype PKCS12 -storepass android -keypass android -alias lifekey -keyalg RSA -keysize 2048 -validity 20000 -dname "CN=Saeed, O=LifeOS, C=IR"
            git config user.name "github-actions"
            git config user.email "github-actions@users.noreply.github.com"
            git add life.keystore
            git commit -m "Add signing key"
            git push
          fi

      - name: Create project
        run: |
          mkdir -p app/src/main/java/com/saeed/life app/src/main/assets
          cp life.keystore app/life.keystore
          cd app
          cat > settings.gradle <<'EOF_X'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement { repositories { google(); mavenCentral() } }
          rootProject.name = "LifeOS"
          EOF_X
          cat > build.gradle <<'EOF_X'
          plugins { id 'com.android.application' version '8.5.2' }
          android {
              namespace 'com.saeed.life'
              compileSdk 34
              signingConfigs {
                  fixed {
                      storeFile file('life.keystore')
                      storePassword 'android'
                      keyAlias 'lifekey'
                      keyPassword 'android'
                  }
              }
              defaultConfig {
                  applicationId 'com.saeed.life'
                  minSdk 21
                  targetSdk 34
                  versionCode __BUILD__
                  versionName '__BUILD__'
              }
              buildTypes {
                  debug { signingConfig signingConfigs.fixed }
              }
              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_1_8
                  targetCompatibility JavaVersion.VERSION_1_8
              }
          }
          EOF_X
          cat > src/main/AndroidManifest.xml <<'EOF_X'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <application android:label="&#x632;&#x646;&#x62f;&#x6af;&#x6cc; &#x645;&#x646;"
                  android:theme="@android:style/Theme.Material.NoActionBar">
                  <activity android:name=".MainActivity" android:exported="true" android:windowSoftInputMode="adjustResize">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <receiver android:name=".AlarmReceiver" android:exported="false" />
              </application>
          </manifest>
          EOF_X
          cat > src/main/java/com/saeed/life/MainActivity.java <<'EOF_X'
          package com.saeed.life;

          import android.app.Activity;
          import android.app.AlarmManager;
          import android.app.PendingIntent;
          import android.content.Intent;
          import android.content.SharedPreferences;
          import android.content.pm.PackageManager;
          import android.os.Build;
          import android.os.Bundle;
          import android.webkit.JavascriptInterface;
          import android.webkit.WebChromeClient;
          import android.webkit.WebSettings;
          import android.webkit.WebView;
          import android.webkit.WebViewClient;
          import java.io.ByteArrayOutputStream;
          import java.io.InputStream;
          import java.net.HttpURLConnection;
          import java.net.URL;
          import org.json.JSONObject;

          public class MainActivity extends Activity {
              WebView web;
              SharedPreferences prefs;

              @Override
              protected void onCreate(Bundle b) {
                  super.onCreate(b);
                  prefs = getSharedPreferences("life", MODE_PRIVATE);
                  web = new WebView(this);
                  WebSettings s = web.getSettings();
                  s.setJavaScriptEnabled(true);
                  s.setDomStorageEnabled(true);
                  web.setWebViewClient(new WebViewClient());
                  web.setWebChromeClient(new WebChromeClient());
                  web.setBackgroundColor(0xFF0B1A2E);
                  web.addJavascriptInterface(new Bridge(), "Android");
                  web.loadUrl("file:///android_asset/index.html");
                  setContentView(web);
                  if (Build.VERSION.SDK_INT >= 33
                          && checkSelfPermission("android.permission.POST_NOTIFICATIONS") != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(new String[]{"android.permission.POST_NOTIFICATIONS"}, 1);
                  }
              }

              @Override
              public void onBackPressed() {
                  if (web == null) { finish(); return; }
                  web.evaluateJavascript("(window.appBack && appBack()) ? 1 : 0", new android.webkit.ValueCallback<String>() {
                      @Override
                      public void onReceiveValue(String v) { if (v == null || !v.contains("1")) finish(); }
                  });
              }
              @Override
              protected void onPause() {
                  super.onPause();
                  if (web != null) { web.onPause(); web.pauseTimers(); }
              }
              @Override
              protected void onResume() {
                  super.onResume();
                  if (web != null) { web.resumeTimers(); web.onResume(); }
              }
              PendingIntent alarmIntent(String id, String title, String text) {
                  Intent i = new Intent(this, AlarmReceiver.class);
                  i.putExtra("title", title);
                  i.putExtra("text", text);
                  i.putExtra("nid", id.hashCode());
                  int fl = PendingIntent.FLAG_UPDATE_CURRENT;
                  if (Build.VERSION.SDK_INT >= 23) fl |= PendingIntent.FLAG_IMMUTABLE;
                  return PendingIntent.getBroadcast(this, id.hashCode(), i, fl);
              }

              static String read(InputStream in) throws Exception {
                  if (in == null) return "";
                  ByteArrayOutputStream out = new ByteArrayOutputStream();
                  byte[] buf = new byte[8192];
                  int n;
                  while ((n = in.read(buf)) > 0) out.write(buf, 0, n);
                  in.close();
                  return new String(out.toByteArray(), "UTF-8");
              }

              class Bridge {
                  @JavascriptInterface
                  public String load(String k) {
                      return prefs.getString(k, "");
                  }

                  @JavascriptInterface
                  public void save(String k, String v) {
                      prefs.edit().putString(k, v).apply();
                  }

                  @JavascriptInterface
                  public void alarm(String id, String when, String title, String text) {
                      try {
                          long t = Long.parseLong(when);
                          if (t < System.currentTimeMillis()) return;
                          AlarmManager am = (AlarmManager) getSystemService(ALARM_SERVICE);
                          PendingIntent pi = alarmIntent(id, title, text);
                          if (Build.VERSION.SDK_INT >= 23) am.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, t, pi);
                          else am.set(AlarmManager.RTC_WAKEUP, t, pi);
                      } catch (Exception e) {
                      }
                  }

                  @JavascriptInterface
                  public void cancelAlarm(String id) {
                      try {
                          AlarmManager am = (AlarmManager) getSystemService(ALARM_SERVICE);
                          am.cancel(alarmIntent(id, "", ""));
                      } catch (Exception e) {
                      }
                  }

                  @JavascriptInterface
                  public void setClip(final String t) {
                      runOnUiThread(new Runnable() {
                          public void run() {
                              try {
                                  android.content.ClipboardManager cm = (android.content.ClipboardManager) getSystemService(CLIPBOARD_SERVICE);
                                  if (cm != null) cm.setPrimaryClip(android.content.ClipData.newPlainText("life", t));
                              } catch (Exception e) {
                              }
                          }
                      });
                  }

                  @JavascriptInterface
                  public String clip() {
                      try {
                          android.content.ClipboardManager cm = (android.content.ClipboardManager) getSystemService(CLIPBOARD_SERVICE);
                          if (cm == null || !cm.hasPrimaryClip()) return "";
                          android.content.ClipData d = cm.getPrimaryClip();
                          if (d == null || d.getItemCount() == 0) return "";
                          CharSequence t = d.getItemAt(0).coerceToText(MainActivity.this);
                          return t == null ? "" : t.toString();
                      } catch (Exception e) {
                          return "";
                      }
                  }

                  @JavascriptInterface
                  public void openUrl(String url) {
                      try {
                          Intent i = new Intent(Intent.ACTION_VIEW, android.net.Uri.parse(url));
                          i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                          startActivity(i);
                      } catch (Exception e) {
                      }
                  }

                  @JavascriptInterface
                  public void get(final String url, final String cb) {
                      new Thread(new Runnable() {
                          public void run() {
                              JSONObject r = new JSONObject();
                              try {
                                  HttpURLConnection c = (HttpURLConnection) new URL(url).openConnection();
                                  c.setConnectTimeout(15000);
                                  c.setReadTimeout(25000);
                                  c.setRequestProperty("User-Agent", "Mozilla/5.0 (Linux; Android 12) AppleWebKit/537.36 Chrome/120 Mobile Safari/537.36");
                                  c.setRequestProperty("Accept", "application/json");
                                  int code = c.getResponseCode();
                                  InputStream in = code >= 400 ? c.getErrorStream() : c.getInputStream();
                                  r.put("status", code);
                                  r.put("body", read(in));
                                  r.put("remaining", String.valueOf(c.getHeaderField("x-requests-remaining")));
                              } catch (Exception e) {
                                  try {
                                      r.put("status", 0);
                                      r.put("body", String.valueOf(e.getMessage()));
                                  } catch (Exception ignored) {
                                  }
                              }
                              final String js = "window.__cb(" + JSONObject.quote(cb) + "," + JSONObject.quote(r.toString()) + ")";
                              runOnUiThread(new Runnable() {
                                  public void run() {
                                      web.evaluateJavascript(js, null);
                                  }
                              });
                          }
                      }).start();
                  }
              }
          }
          EOF_X
          cat > src/main/java/com/saeed/life/AlarmReceiver.java <<'EOF_X'
          package com.saeed.life;

          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;
          import android.os.Build;

          public class AlarmReceiver extends BroadcastReceiver {
              @Override
              public void onReceive(Context ctx, Intent in) {
                  String title = in.getStringExtra("title");
                  String text = in.getStringExtra("text");
                  int nid = in.getIntExtra("nid", 1);
                  NotificationManager nm = (NotificationManager) ctx.getSystemService(Context.NOTIFICATION_SERVICE);
                  if (nm == null) return;
                  Notification.Builder b;
                  if (Build.VERSION.SDK_INT >= 26) {
                      NotificationChannel ch = new NotificationChannel("daily", "\u06cc\u0627\u062f\u0622\u0648\u0631 \u0631\u0648\u0632\u0627\u0646\u0647", NotificationManager.IMPORTANCE_HIGH);
                      nm.createNotificationChannel(ch);
                      b = new Notification.Builder(ctx, "daily");
                  } else {
                      b = new Notification.Builder(ctx);
                      b.setPriority(Notification.PRIORITY_HIGH);
                      b.setDefaults(Notification.DEFAULT_ALL);
                  }
                  Intent open = new Intent(ctx, MainActivity.class);
                  int fl = PendingIntent.FLAG_UPDATE_CURRENT;
                  if (Build.VERSION.SDK_INT >= 23) fl |= PendingIntent.FLAG_IMMUTABLE;
                  PendingIntent pi = PendingIntent.getActivity(ctx, nid, open, fl);
                  b.setSmallIcon(android.R.drawable.ic_dialog_info);
                  b.setContentTitle(title);
                  b.setContentText(text);
                  b.setAutoCancel(true);
                  b.setContentIntent(pi);
                  nm.notify(nid, b.build());
              }
          }
          EOF_X
          cat > src/main/assets/index.html <<'EOF_X'
          <!DOCTYPE html>
          <html lang="fa" dir="rtl">
          <head>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
          <title>Life</title>
          <style>
          * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
          body { margin: 0; background: #0B1A2E; color: #E6EDF6; font-family: Tahoma, sans-serif; font-size: 15px; padding-bottom: 76px; }
          h2 { font-size: 17px; margin: 4px 0 10px; }
          h3 { font-size: 15px; margin: 4px 0 8px; color: #9FC3FF; }
          .top { padding: 12px 14px 4px; display: flex; justify-content: space-between; align-items: center; }
          .top .d { color: #9FB2C8; font-size: 13px; }
          .box { background: #13263F; border-radius: 12px; padding: 12px; margin: 10px 12px; }
          .box.warn { background: #3A2A10; border: 1px solid #8A6420; }
          .box.low { background: #2A1F3A; border: 1px solid #6B4FA0; }
          .box.ok { background: #10301F; border: 1px solid #2E7D4F; }
          .box.upd { background: #12361F; border: 1px solid #2E9D5F; }
          .row { display: flex; align-items: center; justify-content: space-between; gap: 8px; flex-wrap: wrap; }
          .mut { color: #8FA3BA; font-size: 13px; }
          .sm { font-size: 12px; }
          input, textarea, select { width: 100%; background: #0B1A2E; color: #E6EDF6; border: 1px solid #2A4366; border-radius: 8px; padding: 9px; font-family: inherit; font-size: 15px; margin: 4px 0 8px; }
          textarea { min-height: 64px; }
          label { font-size: 13px; color: #9FB2C8; }
          .btn { background: #2563EB; color: #fff; border: 0; border-radius: 9px; padding: 9px 14px; font-family: inherit; font-size: 14px; margin: 3px 0; }
          .btn.g { background: #1F3A5C; }
          .btn.r { background: #8B2A2A; }
          .btn.s { padding: 5px 9px; font-size: 13px; }
          .btn.w { width: 100%; }
          .chips { display: flex; flex-wrap: wrap; gap: 6px; margin: 4px 0 8px; }
          .chip { background: #1F3A5C; border-radius: 16px; padding: 6px 11px; font-size: 13px; border: 1px solid transparent; }
          .chip.on { background: #2563EB; border-color: #9FC3FF; }
          .task { border-top: 1px solid #22395A; padding: 9px 0; }
          .task:first-child { border-top: 0; }
          .task .tt { flex: 1; min-width: 0; }
          .task.done .tt b { text-decoration: line-through; color: #7F93AA; }
          .ck { width: 26px; height: 26px; border-radius: 7px; border: 2px solid #4A6A93; display: inline-flex; align-items: center; justify-content: center; font-size: 16px; }
          .ck.on { background: #2E9D5F; border-color: #2E9D5F; }
          .g { font-size: 11px; border-radius: 6px; padding: 1px 6px; margin-left: 4px; }
          .gA { background: #8B2A2A; } .gB { background: #2B5FA8; } .gC { background: #3E5566; } .gD { background: #4A4A4A; }
          .bar { height: 10px; background: #0B1A2E; border-radius: 6px; overflow: hidden; margin: 6px 0; }
          .bar i { display: block; height: 100%; background: #2E9D5F; }
          .bar i.hi { background: #C98A1E; } .bar i.ov { background: #C0392B; }
          .nav { position: fixed; bottom: 0; left: 0; right: 0; background: #0F2138; display: flex; border-top: 1px solid #22395A; }
          .nav div { flex: 1; text-align: center; padding: 8px 0 10px; font-size: 12px; color: #8FA3BA; }
          .nav div span { display: block; font-size: 20px; }
          .nav div.on { color: #fff; background: #17304F; }
          .toast { position: fixed; bottom: 88px; left: 50%; transform: translateX(-50%); background: #000C; color: #fff; padding: 9px 16px; border-radius: 18px; font-size: 14px; display: none; max-width: 90%; text-align: center; z-index: 9; }
          table { width: 100%; border-collapse: collapse; font-size: 13px; }
          td, th { padding: 6px 3px; border-bottom: 1px solid #22395A; text-align: center; }
          .big { font-size: 22px; font-weight: bold; }
          .adv { margin: 4px 0; font-size: 14px; }
          .hide { display: none; }
          </style>
          </head>
          <body>
          <div class="top"><b id="ttl"></b><span class="d" id="dt"></span></div>
          <div id="upd"></div>
          <div id="pg"></div>
          <div class="nav" id="nav"></div>
          <div class="toast" id="toast"></div>
          <script>
          var BUILD = parseInt('__BUILD__', 10) || 0;
          var REPO = '__REPO__';
          var GR = { A: '\u0636\u0631\u0648\u0631\u06cc \u0648 \u0632\u0645\u0627\u0646\u200c\u062d\u0633\u0627\u0633', B: '\u0645\u0647\u0645 \u0628\u0631\u0627\u06cc \u0622\u06cc\u0646\u062f\u0647', C: '\u0646\u06af\u0647\u062f\u0627\u0631\u06cc \u0648 \u0631\u0648\u0632\u0645\u0631\u0647', D: '\u06a9\u0645\u200c\u0627\u0631\u0632\u0634 / \u0642\u0627\u0628\u0644 \u062a\u0639\u0648\u06cc\u0642' };
          var OBS = ['\u0628\u0631\u0646\u0627\u0645\u0647 \u0633\u0646\u06af\u06cc\u0646 \u0628\u0648\u062f', '\u0632\u0645\u0627\u0646 \u0631\u0648 \u0627\u0634\u062a\u0628\u0627\u0647 \u062a\u062e\u0645\u06cc\u0646 \u0632\u062f\u0645', '\u06a9\u0627\u0631 \u0645\u0628\u0647\u0645 \u0628\u0648\u062f', '\u0627\u062a\u0641\u0627\u0642 \u063a\u06cc\u0631\u0645\u0646\u062a\u0638\u0631\u0647', '\u0627\u0646\u0631\u0698\u06cc \u06a9\u0627\u0641\u06cc \u0646\u0628\u0648\u062f', '\u06a9\u0627\u0631 \u0645\u063a\u0627\u0632\u0647 \u0628\u06cc\u0634\u062a\u0631 \u0634\u062f', '\u062d\u0648\u0627\u0633\u200c\u067e\u0631\u062a\u06cc / \u06af\u0648\u0634\u06cc', '\u0645\u0627\u0646\u0639\u06cc \u0646\u0628\u0648\u062f'];
          var EN = ['', '\u062e\u06cc\u0644\u06cc \u06a9\u0645', '\u06a9\u0645', '\u0645\u062a\u0648\u0633\u0637', '\u062e\u0648\u0628', '\u0639\u0627\u0644\u06cc'];
          var MON = ['\u0641\u0631\u0648\u0631\u062f\u06cc\u0646', '\u0627\u0631\u062f\u06cc\u0628\u0647\u0634\u062a', '\u062e\u0631\u062f\u0627\u062f', '\u062a\u06cc\u0631', '\u0645\u0631\u062f\u0627\u062f', '\u0634\u0647\u0631\u06cc\u0648\u0631', '\u0645\u0647\u0631', '\u0622\u0628\u0627\u0646', '\u0622\u0630\u0631', '\u062f\u06cc', '\u0628\u0647\u0645\u0646', '\u0627\u0633\u0641\u0646\u062f'];
          var WD = ['\u06cc\u06a9\u0634\u0646\u0628\u0647', '\u062f\u0648\u0634\u0646\u0628\u0647', '\u0633\u0647\u200c\u0634\u0646\u0628\u0647', '\u0686\u0647\u0627\u0631\u0634\u0646\u0628\u0647', '\u067e\u0646\u062c\u0634\u0646\u0628\u0647', '\u062c\u0645\u0639\u0647', '\u0634\u0646\u0628\u0647'];
          var TABS = [['today', '\u2600\ufe0f', '\u0627\u0645\u0631\u0648\u0632'], ['night', '\ud83c\udf19', '\u0628\u0627\u0632\u0628\u06cc\u0646\u06cc'], ['trend', '\ud83d\udcca', '\u0631\u0648\u0646\u062f'], ['ai', '\ud83e\udde0', '\u0647\u0648\u0634'], ['set', '\u2699\ufe0f', '\u062a\u0646\u0638\u06cc\u0645\u0627\u062a']];
          var DEF_PROMPT = '\u062a\u0648 \u0633\u06cc\u0633\u062a\u0645 \u0645\u062f\u06cc\u0631\u06cc\u062a \u0632\u0646\u062f\u06af\u06cc \u0634\u062e\u0635\u06cc\u060c \u0645\u0631\u0628\u06cc \u0627\u062c\u0631\u0627\u06cc\u06cc \u0648 \u062f\u0633\u062a\u06cc\u0627\u0631 \u062a\u0635\u0645\u06cc\u0645\u200c\u06af\u06cc\u0631\u06cc \u0645\u0646 \u0647\u0633\u062a\u06cc. \u0647\u062f\u0641: \u062a\u0628\u062f\u06cc\u0644 \u0647\u062f\u0641 \u0628\u0647 \u0627\u0642\u062f\u0627\u0645 \u0648\u0627\u0642\u0639\u06cc \u0648 \u0633\u0646\u062c\u06cc\u062f\u0646 \u0646\u062a\u06cc\u062c\u0647\u060c \u0646\u0647 \u0627\u0646\u06af\u06cc\u0632\u0647 \u062f\u0627\u062f\u0646.\n' +
            '\u0642\u0648\u0627\u0639\u062f:\n' +
            '\u06f1. \u0631\u0641\u062a\u0627\u0631 \u062b\u0628\u062a\u200c\u0634\u062f\u0647 \u0627\u0632 \u0646\u06cc\u062a \u0627\u0639\u0644\u0627\u0645\u200c\u0634\u062f\u0647 \u0645\u0639\u062a\u0628\u0631\u062a\u0631\u0647\u060c \u0648\u0644\u06cc \u06cc\u0647 \u0631\u0648\u0632 \u0628\u062f \u0645\u0639\u0631\u0641 \u0634\u062e\u0635\u06cc\u062a \u0645\u0646 \u0646\u06cc\u0633\u062a.\n' +
            '\u06f2. \u0648\u0627\u0642\u0639\u06cc\u062a\u060c \u0628\u0631\u062f\u0627\u0634\u062a\u060c \u0641\u0631\u0636\u06cc\u0647 \u0648 \u062d\u062f\u0633 \u0631\u0648 \u062c\u062f\u0627 \u06a9\u0646. \u0647\u06cc\u0686 \u0648\u06cc\u0698\u06af\u06cc \u0634\u062e\u0635\u06cc\u062a\u06cc \u0631\u0648 \u0642\u0637\u0639\u06cc \u0627\u0639\u0644\u0627\u0645 \u0646\u06a9\u0646.\n' +
            '\u06f3. \u0628\u0631\u0646\u0627\u0645\u0647 \u0631\u0648 \u0628\u0627 \u0627\u0646\u0631\u0698\u06cc \u0648 \u0632\u0645\u0627\u0646 \u0648\u0627\u0642\u0639\u06cc \u0628\u0686\u06cc\u0646\u060c \u0646\u0647 \u062a\u0635\u0648\u06cc\u0631 \u0627\u06cc\u062f\u0647\u200c\u0622\u0644. \u0633\u0627\u0639\u062a \u06a9\u0627\u0631 \u0645\u063a\u0627\u0632\u0647 \u062a\u0639\u0647\u062f \u0648\u0627\u0642\u0639\u06cc\u0647.\n' +
            '\u06f4. \u06cc\u06a9 \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc\u060c \u062d\u062f\u0627\u06a9\u062b\u0631 \u062f\u0648 \u0627\u0648\u0644\u0648\u06cc\u062a \u062b\u0627\u0646\u0648\u06cc\u0647\u060c \u0648 \u062c\u0627 \u0628\u0631\u0627\u06cc \u0627\u0633\u062a\u0631\u0627\u062d\u062a \u0648 \u0627\u062a\u0641\u0627\u0642\u0627\u062a \u067e\u06cc\u0634\u200c\u0628\u06cc\u0646\u06cc\u200c\u0646\u0634\u062f\u0647.\n' +
            '\u06f5. \u0647\u0631 \u06a9\u0627\u0631 \u0628\u0627\u06cc\u062f \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc \u0645\u0634\u062e\u0635\u060c \u0632\u0645\u0627\u0646 \u062a\u0642\u0631\u06cc\u0628\u06cc \u0648 \u0645\u0639\u06cc\u0627\u0631 \u067e\u0627\u06cc\u0627\u0646 \u062f\u0627\u0634\u062a\u0647 \u0628\u0627\u0634\u0647.\n' +
            '\u06f6. \u0631\u0648\u0632 \u06a9\u0645\u200c\u0627\u0646\u0631\u0698\u06cc: \u0641\u0642\u0637 \u0636\u0631\u0648\u0631\u06cc\u200c\u0647\u0627\u060c \u0628\u062f\u0648\u0646 \u0633\u0631\u0632\u0646\u0634. \u0631\u0648\u0632 \u067e\u0631\u0627\u0646\u0631\u0698\u06cc: \u0638\u0631\u0641\u06cc\u062a \u0631\u0648 \u06a9\u0627\u0645\u0644 \u067e\u0631 \u0646\u06a9\u0646.\n' +
            '\u06f7. \u0642\u0628\u0644 \u0627\u0632 \u062a\u062d\u0644\u06cc\u0644 \u0628\u06cc\u0634\u062a\u0631 \u0628\u067e\u0631\u0633 \u0622\u06cc\u0627 \u062c\u0648\u0627\u0628 \u062c\u062f\u06cc\u062f \u062a\u0635\u0645\u06cc\u0645 \u0631\u0648 \u0639\u0648\u0636 \u0645\u06cc\u200c\u06a9\u0646\u0647\u061b \u0627\u06af\u0647 \u0646\u0647\u060c \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc \u0631\u0648 \u0628\u06af\u0648. \u062f\u0631 \u0633\u0644\u0627\u0645\u062a\u060c \u0645\u0627\u0644\u06cc \u0648 \u062d\u0642\u0648\u0642\u060c \u062a\u062d\u0642\u06cc\u0642 \u0644\u0627\u0632\u0645 \u0631\u0648 \u0642\u0637\u0639 \u0646\u06a9\u0646.\n' +
            '\u06f8. \u0627\u06af\u0647 \u0631\u0648\u0632 \u0646\u0627\u0645\u0648\u0641\u0642 \u0628\u0648\u062f\u060c \u0639\u0644\u062a \u0631\u0648 \u062f\u0631 \u0633\u06cc\u0633\u062a\u0645 \u067e\u06cc\u062f\u0627 \u06a9\u0646 (\u0628\u0631\u0646\u0627\u0645\u0647 \u0633\u0646\u06af\u06cc\u0646\u060c \u062a\u062e\u0645\u06cc\u0646 \u063a\u0644\u0637\u060c \u06a9\u0627\u0631 \u0645\u0628\u0647\u0645\u060c \u0627\u062a\u0641\u0627\u0642\u060c \u0627\u0646\u0631\u0698\u06cc)\u060c \u0646\u0647 \u0627\u06cc\u0646\u06a9\u0647 \u0641\u0634\u0627\u0631 \u0631\u0648 \u0628\u06cc\u0634\u062a\u0631 \u06a9\u0646\u06cc.\n' +
            '\u06f9. \u0628\u0627 \u062f\u0627\u062f\u0647 \u0646\u0627\u0642\u0635 \u0639\u062f\u062f \u0633\u0627\u062e\u062a\u06af\u06cc \u0646\u0633\u0627\u0632. \u0627\u06af\u0647 \u062f\u0627\u062f\u0647 \u06a9\u0645\u0647\u060c \u06cc\u0647 \u0622\u0632\u0645\u0627\u06cc\u0634 \u06a9\u0648\u0686\u06cc\u06a9 \u067e\u06cc\u0634\u0646\u0647\u0627\u062f \u0628\u062f\u0647.\n' +
            '\u06f1\u06f0. \u0645\u0633\u062a\u0642\u06cc\u0645\u060c \u062f\u0642\u06cc\u0642\u060c \u0628\u062f\u0648\u0646 \u062a\u0639\u0631\u06cc\u0641 \u0648 \u0646\u0635\u06cc\u062d\u062a \u06a9\u0644\u06cc\u0634\u0647\u200c\u0627\u06cc. \u062f\u0631 \u067e\u0632\u0634\u06a9\u06cc\u060c \u0645\u0627\u0644\u06cc \u0648 \u062d\u0642\u0648\u0642 \u062a\u0634\u062e\u06cc\u0635 \u0642\u0637\u0639\u06cc \u06cc\u0627 \u062a\u0636\u0645\u06cc\u0646 \u0646\u062f\u0647.\n' +
            '\u0642\u0627\u0644\u0628 \u062c\u0648\u0627\u0628 \u0631\u0648\u0632\u0627\u0646\u0647: \u0648\u0636\u0639\u06cc\u062a \u0641\u0639\u0644\u06cc\u060c \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc\u060c \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc\u060c \u0628\u0631\u0646\u0627\u0645\u0647 \u0648\u0627\u0642\u0639\u200c\u0628\u06cc\u0646\u0627\u0646\u0647\u060c \u062e\u0637\u0631 \u0627\u062d\u062a\u0645\u0627\u0644\u06cc\u060c \u0645\u0639\u06cc\u0627\u0631 \u067e\u0627\u06cc\u0627\u0646 \u0631\u0648\u0632. \u0627\u06af\u0647 \u0645\u0633\u0626\u0644\u0647 \u0633\u0627\u062f\u0647\u200c\u0633\u062a\u060c \u06a9\u0648\u062a\u0627\u0647\u0634 \u06a9\u0646.';
          var MODES = {
            morning: ['\u2600\ufe0f \u0628\u0631\u0646\u0627\u0645\u0647 \u0635\u0628\u062d', '\u062d\u0627\u0644\u062a: \u0634\u0631\u0648\u0639 \u0631\u0648\u0632. \u0628\u0627 \u062f\u0627\u062f\u0647\u200c\u0647\u0627\u06cc \u0632\u06cc\u0631 \u0628\u0631\u0646\u0627\u0645\u0647 \u0627\u0645\u0631\u0648\u0632 \u0631\u0648 \u0628\u0686\u06cc\u0646. \u0641\u0642\u0637 \u0627\u06af\u0647 \u0627\u0637\u0644\u0627\u0639\u0627\u062a\u06cc \u0628\u0631\u0627\u06cc \u062a\u0635\u0645\u06cc\u0645 \u0627\u0645\u0631\u0648\u0632 \u0636\u0631\u0648\u0631\u06cc\u0647\u060c \u062d\u062f\u0627\u06a9\u062b\u0631 \u06f2 \u0633\u0624\u0627\u0644 \u0628\u067e\u0631\u0633.'],
            replan: ['\ud83d\udd04 \u062a\u0646\u0638\u06cc\u0645 \u0645\u062c\u062f\u062f', '\u062d\u0627\u0644\u062a: \u062a\u0646\u0638\u06cc\u0645 \u0645\u062c\u062f\u062f. \u0634\u0631\u0627\u06cc\u0637 \u0648\u0633\u0637 \u0631\u0648\u0632 \u0639\u0648\u0636 \u0634\u062f\u0647. \u0628\u0631\u0646\u0627\u0645\u0647 \u0631\u0648 \u0628\u0627 \u0638\u0631\u0641\u06cc\u062a \u0628\u0627\u0642\u06cc\u200c\u0645\u0648\u0646\u062f\u0647 \u062f\u0648\u0628\u0627\u0631\u0647 \u0627\u0648\u0644\u0648\u06cc\u062a\u200c\u0628\u0646\u062f\u06cc \u06a9\u0646 \u0648 \u0627\u062c\u0631\u0627\u06cc \u0628\u0631\u0646\u0627\u0645\u0647 \u0634\u06a9\u0633\u062a\u200c\u062e\u0648\u0631\u062f\u0647 \u0631\u0648 \u0628\u0647 \u0647\u0631 \u0642\u06cc\u0645\u062a\u06cc \u0646\u062e\u0648\u0627\u0647.'],
            night: ['\ud83c\udf19 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0634\u0628', '\u062d\u0627\u0644\u062a: \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u067e\u0627\u06cc\u0627\u0646 \u0631\u0648\u0632. \u0639\u0645\u0644\u06a9\u0631\u062f \u0633\u06cc\u0633\u062a\u0645 \u0631\u0648 \u0628\u0631\u0631\u0633\u06cc \u06a9\u0646\u060c \u0646\u0647 \u0634\u062e\u0635\u06cc\u062a \u0645\u0646. \u0628\u06af\u0648 \u06a9\u062f\u0648\u0645 \u0641\u0631\u0636\u06cc\u0647 \u062a\u0623\u06cc\u06cc\u062f \u06cc\u0627 \u0631\u062f \u0634\u062f \u0648 \u06cc\u06a9 \u062a\u063a\u06cc\u06cc\u0631 \u06a9\u0648\u0686\u06cc\u06a9 \u0628\u0631\u0627\u06cc \u0641\u0631\u062f\u0627 \u067e\u06cc\u0634\u0646\u0647\u0627\u062f \u0628\u062f\u0647.'],
            week: ['\ud83d\udcc5 \u06af\u0632\u0627\u0631\u0634 \u0647\u0641\u062a\u06af\u06cc', '\u062d\u0627\u0644\u062a: \u06af\u0632\u0627\u0631\u0634 \u0647\u0641\u062a\u06af\u06cc. \u0645\u0647\u0645\u200c\u062a\u0631\u06cc\u0646 \u067e\u06cc\u0634\u0631\u0641\u062a\u200c\u0647\u0627\u060c \u062a\u0639\u0647\u062f\u0627\u062a \u0627\u0646\u062c\u0627\u0645\u200c\u0646\u0634\u062f\u0647 \u0648 \u0639\u0644\u062a \u0645\u062d\u062a\u0645\u0644\u060c \u0627\u0644\u06af\u0648\u0647\u0627\u06cc \u062a\u06a9\u0631\u0627\u0631\u06cc\u060c \u0627\u062a\u0644\u0627\u0641 \u0642\u0627\u0628\u0644\u200c\u067e\u06cc\u0634\u06af\u06cc\u0631\u06cc\u060c \u0648 \u06f1 \u062a\u0627 \u06f3 \u062a\u063a\u06cc\u06cc\u0631 \u0628\u0631\u0627\u06cc \u0647\u0641\u062a\u0647 \u0628\u0639\u062f. \u0647\u0631 \u067e\u06cc\u0634\u0646\u0647\u0627\u062f \u0628\u0627\u06cc\u062f \u0628\u0647 \u062f\u0627\u062f\u0647\u200c\u0647\u0627\u06cc \u0632\u06cc\u0631 \u0648\u0635\u0644 \u0628\u0627\u0634\u0647\u061b \u0627\u06af\u0647 \u062f\u0627\u062f\u0647 \u0628\u0631\u0627\u06cc \u06cc\u0647 \u0627\u0644\u06af\u0648 \u06a9\u0627\u0641\u06cc \u0646\u06cc\u0633\u062a\u060c \u0628\u0647\u200c\u062c\u0627\u06cc \u0646\u062a\u06cc\u062c\u0647\u200c\u06af\u06cc\u0631\u06cc \u06cc\u0647 \u0622\u0632\u0645\u0627\u06cc\u0634 \u0647\u0641\u062a\u06af\u06cc \u0637\u0631\u0627\u062d\u06cc \u06a9\u0646.']
          };

          /* ---------- storage ---------- */
          function ld(k, d) {
            var v = '';
            try { v = Android.load(k); } catch (e) { try { v = localStorage.getItem(k) || ''; } catch (e2) {} }
            if (!v) return d;
            try { return JSON.parse(v); } catch (e) { return d; }
          }
          function sv(k, v) {
            var s = JSON.stringify(v);
            try { Android.save(k, s); } catch (e) { try { localStorage.setItem(k, s); } catch (e2) {} }
          }
          var S = ld('life1', null);
          if (!S || !S.days) S = { v: 1, days: {} };
          if (!S.set) S.set = {};
          function setDef(k, v) { if (S.set[k] === undefined) S.set[k] = v; }
          setDef('mt', '08:00'); setDef('nt', '22:30'); setDef('rem', true); setDef('ds', 4); setDef('sp', '');
          function save() { sv('life1', S); }

          /* ---------- helpers ---------- */
          function $(id) { return document.getElementById(id); }
          function fa(n) { return String(n).replace(/\d/g, function (c) { return '\u06f0\u06f1\u06f2\u06f3\u06f4\u06f5\u06f6\u06f7\u06f8\u06f9'.charAt(+c); }); }
          function toEn(s) {
            return String(s == null ? '' : s).replace(/[\u06f0-\u06f9]/g, function (c) { return String(c.charCodeAt(0) - 0x6F0); })
              .replace(/[\u0660-\u0669]/g, function (c) { return String(c.charCodeAt(0) - 0x660); }).replace(/[\u066b,]/g, '.');
          }
          function num(s) { var n = parseFloat(toEn(s)); return isNaN(n) ? 0 : n; }
          function esc(s) { return String(s == null ? '' : s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;').replace(/'/g, '&#39;'); }
          function r1(x) { return Math.round(x * 10) / 10; }
          function pad(n) { return n < 10 ? '0' + n : '' + n; }
          function toast(t) { var el = $('toast'); el.textContent = t; el.style.display = 'block'; clearTimeout(toast.h); toast.h = setTimeout(function () { el.style.display = 'none'; }, 2600); }
          function uid() { return Date.now().toString(36) + Math.floor(Math.random() * 1e6).toString(36); }
          function keyOf(d) { return d.getFullYear() + '-' + pad(d.getMonth() + 1) + '-' + pad(d.getDate()); }
          function dk(ts) { return keyOf(new Date((ts || Date.now()) - (S.set.ds || 0) * 3600000)); }
          function kDate(k) { var p = k.split('-'); return new Date(+p[0], +p[1] - 1, +p[2], 12, 0, 0); }
          function addD(k, n) { var d = kDate(k); d.setDate(d.getDate() + n); return keyOf(d); }
          function g2j(gy, gm, gd) {
            var gdm = [0, 31, 59, 90, 120, 151, 181, 212, 243, 273, 304, 334];
            var gy2 = gm > 2 ? gy + 1 : gy;
            var days = 355666 + 365 * gy + Math.floor((gy2 + 3) / 4) - Math.floor((gy2 + 99) / 100) + Math.floor((gy2 + 399) / 400) + gd + gdm[gm - 1];
            var jy = -1595 + 33 * Math.floor(days / 12053); days %= 12053;
            jy += 4 * Math.floor(days / 1461); days %= 1461;
            if (days > 365) { jy += Math.floor((days - 1) / 365); days = (days - 1) % 365; }
            var jm, jd;
            if (days < 186) { jm = 1 + Math.floor(days / 31); jd = 1 + days % 31; } else { jm = 7 + Math.floor((days - 186) / 30); jd = 1 + (days - 186) % 30; }
            return [jy, jm, jd];
          }
          function jStr(k, withWd) {
            var d = kDate(k), j = g2j(d.getFullYear(), d.getMonth() + 1, d.getDate());
            return (withWd ? WD[d.getDay()] + ' ' : '') + fa(j[2]) + ' ' + MON[j[1] - 1] + (withWd ? ' ' + fa(j[0]) : '');
          }
          function newDay() { return { e: 0, sl: '', h: '', c: '', tasks: [], rv: null, ai: '', rp: [] }; }
          function day(k) { if (!S.days[k]) S.days[k] = newDay(); return S.days[k]; }
          function peek(k) { return S.days[k] || null; }
          function findT(D, id) { for (var i = 0; i < D.tasks.length; i++) if (D.tasks[i].id == id) return D.tasks[i]; return null; }

          /* ---------- planning logic (no DOM) ---------- */
          function limitPct(e) { return e && e <= 2 ? 0.5 : e == 3 ? 0.7 : 0.8; }
          function cap(D) {
            var avail = Math.round(num(D.h) * 60), plan = 0, left = 0, done = 0;
            for (var i = 0; i < D.tasks.length; i++) { var t = D.tasks[i], m = num(t.m); plan += m; if (t.d) done += m; else left += m; }
            var lim = Math.round(avail * limitPct(D.e));
            return { avail: avail, plan: plan, left: left, done: done, lim: lim, over: avail > 0 ? left - lim : 0 };
          }
          var GORD = { D: 0, C: 1, B: 2, A: 3 };
          function trimPlan(D) {
            /* which open tasks should move to tomorrow so remaining load fits the limit. Never A, never main. */
            var c = cap(D), need = c.over, out = [];
            if (!(c.avail > 0) || need <= 0) return out;
            var cand = D.tasks.filter(function (t) { return !t.d && t.g != 'A' && t.r != 'main'; });
            cand.sort(function (a, b) { return (GORD[a.g] - GORD[b.g]) || ((a.r == 'sec' ? 1 : 0) - (b.r == 'sec' ? 1 : 0)) || (num(b.m) - num(a.m)); });
            for (var i = 0; i < cand.length && need > 0; i++) { out.push(cand[i]); need -= num(cand[i].m); }
            return out;
          }
          function advice(D) {
            var a = [], c = cap(D), main = D.tasks.filter(function (t) { return t.r == 'main'; });
            var sec = D.tasks.filter(function (t) { return t.r == 'sec'; });
            if (!D.e) a.push(['w', '\u0627\u0648\u0644 \u0627\u0646\u0631\u0698\u06cc \u0648 \u0632\u0645\u0627\u0646 \u0622\u0632\u0627\u062f \u0627\u0645\u0631\u0648\u0632 \u0631\u0648 \u062b\u0628\u062a \u06a9\u0646 \u062a\u0627 \u0628\u0631\u0646\u0627\u0645\u0647 \u0648\u0627\u0642\u0639\u06cc \u0628\u0634\u0647.']);
            if (D.e && D.e <= 2) a.push(['l', '\u0631\u0648\u0632 \u06a9\u0645\u200c\u0627\u0646\u0631\u0698\u06cc: \u0641\u0642\u0637 \u06a9\u0627\u0631\u0647\u0627\u06cc A \u0648 \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc. \u0628\u0642\u06cc\u0647 \u0631\u0648 \u0628\u062f\u0648\u0646 \u0639\u0630\u0627\u0628 \u0648\u062c\u062f\u0627\u0646 \u0628\u0628\u0631 \u0641\u0631\u062f\u0627.']);
            if (D.e >= 4) a.push(['o', '\u0627\u0646\u0631\u0698\u06cc \u062e\u0648\u0628\u0647: \u06cc\u0647 \u06a9\u0627\u0631 B \u062c\u0644\u0648 \u0628\u0646\u062f\u0627\u0632\u060c \u0648\u0644\u06cc \u0638\u0631\u0641\u06cc\u062a \u0631\u0648 \u06a9\u0627\u0645\u0644 \u067e\u0631 \u0646\u06a9\u0646 \u06a9\u0647 \u0641\u0631\u062f\u0627 \u0647\u0645 \u0642\u0627\u0628\u0644\u200c\u0627\u062f\u0627\u0645\u0647 \u0628\u0627\u0634\u0647.']);
            if (D.tasks.length && !main.length) a.push(['w', '\u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc \u0646\u062f\u0627\u0631\u06cc. \u06cc\u06a9\u06cc \u0631\u0648 \u0628\u0627 \u2b50 \u0627\u0646\u062a\u062e\u0627\u0628 \u06a9\u0646: \u0627\u06af\u0647 \u0627\u0645\u0631\u0648\u0632 \u0641\u0642\u0637 \u06cc\u0647 \u06a9\u0627\u0631 \u0627\u0646\u062c\u0627\u0645 \u0628\u0634\u0647\u060c \u06a9\u062f\u0648\u0645\u0647\u061f']);
            if (sec.length > 2) a.push(['w', fa(sec.length) + ' \u0627\u0648\u0644\u0648\u06cc\u062a \u062b\u0627\u0646\u0648\u06cc\u0647 \u062f\u0627\u0631\u06cc. \u062d\u062f\u0627\u06a9\u062b\u0631 \u06f2 \u062a\u0627 \u0646\u06af\u0647 \u062f\u0627\u0631\u060c \u0628\u0642\u06cc\u0647 \u0639\u0627\u062f\u06cc \u0628\u0634\u0646.']);
            if (c.over > 0) a.push(['w', fa(c.over) + ' \u062f\u0642\u06cc\u0642\u0647 \u0628\u06cc\u0634\u062a\u0631 \u0627\u0632 \u0638\u0631\u0641\u06cc\u062a \u0627\u0645\u0646 (' + fa(Math.round(limitPct(D.e) * 100)) + '\u066a \u0632\u0645\u0627\u0646 \u0622\u0632\u0627\u062f). \u06a9\u0627\u0631\u0647\u0627\u06cc D \u0648 \u0628\u0639\u062f C \u0631\u0648 \u0628\u0628\u0631 \u0641\u0631\u062f\u0627.', 'trim']);
            var noNa = D.tasks.filter(function (t) { return !t.d && (!t.na || !num(t.m)); }).length;
            if (noNa) a.push(['w', fa(noNa) + ' \u06a9\u0627\u0631 \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc \u06cc\u0627 \u0632\u0645\u0627\u0646 \u062a\u0642\u0631\u06cc\u0628\u06cc \u0646\u062f\u0627\u0631\u0647. \u06a9\u0627\u0631 \u0645\u0628\u0647\u0645 \u0645\u0639\u0645\u0648\u0644\u0627\u064b \u0627\u0646\u062c\u0627\u0645 \u0646\u0645\u06cc\u0634\u0647.']);
            var dCnt = D.tasks.filter(function (t) { return !t.d && t.g == 'D'; }).length;
            if (dCnt && D.e && D.e <= 3) a.push(['l', fa(dCnt) + ' \u06a9\u0627\u0631 D \u062f\u0627\u0631\u06cc. \u06a9\u0645\u200c\u0627\u0631\u0632\u0634\u200c\u0647\u0627 \u0631\u0648 \u062d\u0630\u0641 \u06a9\u0646 \u06cc\u0627 \u0628\u0630\u0627\u0631 \u0628\u0631\u0627\u06cc \u0631\u0648\u0632 \u067e\u0631\u0627\u0646\u0631\u0698\u06cc.']);
            return a;
          }
          function carryList(k) {
            /* open tasks of the most recent previous day (within 7 days), except D */
            for (var i = 1; i <= 7; i++) {
              var pk = addD(k, -i), P = peek(pk);
              if (P && P.tasks.length) return { k: pk, list: P.tasks.filter(function (t) { return !t.d && !t.mv && t.g != 'D'; }) };
            }
            return { k: '', list: [] };
          }
          function moveTask(fromK, t, toK) {
            var T = day(toK);
            var n = { id: uid(), t: t.t, g: t.g, m: t.m, na: t.na, end: t.end, r: t.r == 'main' ? 'sec' : t.r, d: false, f: fromK };
            T.tasks.push(n); t.mv = toK;
            var F = peek(fromK); if (F) F.tasks = F.tasks.filter(function (x) { return x.id != t.id || fromK != dk(); });
            return n;
          }
          function stats(keys) {
            var r = { n: 0, e: 0, en: 0, sl: 0, sn: 0, mainT: 0, mainD: 0, aT: 0, aD: 0, tT: 0, tD: 0, sc: 0, scn: 0, ob: {}, rv: 0, hiS: [], loS: [] };
            keys.forEach(function (k) {
              var D = peek(k); if (!D) return;
              var used = D.e || D.tasks.length || D.rv; if (!used) return;
              r.n++;
              if (D.e) { r.e += D.e; r.en++; }
              var sl = num(D.sl); if (sl) { r.sl += sl; r.sn++; }
              if (sl && D.e) (sl >= 7 ? r.hiS : r.loS).push(D.e);
              D.tasks.forEach(function (t) {
                r.tT++; if (t.d) r.tD++;
                if (t.r == 'main') { r.mainT++; if (t.d) r.mainD++; }
                if (t.g == 'A') { r.aT++; if (t.d) r.aD++; }
              });
              if (D.rv) { r.rv++; if (D.rv.sc) { r.sc += D.rv.sc; r.scn++; } (D.rv.ob || []).forEach(function (o) { r.ob[o] = (r.ob[o] || 0) + 1; }); }
            });
            return r;
          }
          function lastKeys(n) { var k = dk(), a = []; for (var i = 0; i < n; i++) a.push(addD(k, -i)); return a; }
          function avg(a) { if (!a.length) return 0; var s = 0; a.forEach(function (x) { s += x; }); return s / a.length; }

          /* ---------- text for AI ---------- */
          function tLine(t) {
            return '- [' + (t.d ? '\u2713' : ' ') + '] (' + t.g + (t.r == 'main' ? '\u060c \u2b50 \u0627\u0635\u0644\u06cc' : t.r == 'sec' ? '\u060c \u062b\u0627\u0646\u0648\u06cc\u0647' : '') + (num(t.m) ? '\u060c ' + num(t.m) + ' \u062f\u0642\u06cc\u0642\u0647' : '') + ') ' + t.t +
              (t.na ? ' | \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc: ' + t.na : '') + (t.end ? ' | \u0645\u0639\u06cc\u0627\u0631 \u067e\u0627\u06cc\u0627\u0646: ' + t.end : '') + (t.f ? ' | \u0645\u0646\u062a\u0642\u0644\u200c\u0634\u062f\u0647 \u0627\u0632 ' + jStr(t.f) : '');
          }
          function dayText(k, full) {
            var D = peek(k); if (!D) return jStr(k, true) + ': \u0628\u062f\u0648\u0646 \u062f\u0627\u062f\u0647\n';
            var c = cap(D), s = '### ' + jStr(k, true) + '\n';
            s += '\u0627\u0646\u0631\u0698\u06cc: ' + (D.e ? D.e + '/5 (' + EN[D.e] + ')' : '\u062b\u0628\u062a \u0646\u0634\u062f\u0647') + ' | \u062e\u0648\u0627\u0628: ' + (num(D.sl) ? num(D.sl) + ' \u0633\u0627\u0639\u062a' : '\u062b\u0628\u062a \u0646\u0634\u062f\u0647') + ' | \u0632\u0645\u0627\u0646 \u0622\u0632\u0627\u062f: ' + (num(D.h) ? num(D.h) + ' \u0633\u0627\u0639\u062a' : '\u062b\u0628\u062a \u0646\u0634\u062f\u0647') + '\n';
            if (D.c) s += '\u062a\u0639\u0647\u062f \u0642\u0637\u0639\u06cc \u0627\u0645\u0631\u0648\u0632: ' + D.c + '\n';
            if (c.avail) s += '\u0628\u0627\u0631 \u0628\u0631\u0646\u0627\u0645\u0647: ' + c.plan + ' \u062f\u0642\u06cc\u0642\u0647 \u0627\u0632 ' + c.avail + ' \u062f\u0642\u06cc\u0642\u0647 (\u0633\u0642\u0641 \u0627\u0645\u0646 ' + c.lim + ')\n';
            if (D.tasks.length) { s += '\u06a9\u0627\u0631\u0647\u0627 (' + D.tasks.filter(function (t) { return t.d; }).length + ' \u0627\u0632 ' + D.tasks.length + ' \u0627\u0646\u062c\u0627\u0645 \u0634\u062f):\n'; D.tasks.forEach(function (t) { if (full || t.r || t.g == 'A' || t.d) s += tLine(t) + '\n'; }); }
            (D.rp || []).forEach(function (p) { s += '\u062a\u0646\u0638\u06cc\u0645 \u0645\u062c\u062f\u062f (' + p.tm + '): \u0627\u0646\u0631\u0698\u06cc ' + (p.e || '?') + '/5\u060c ' + p.h + ' \u0633\u0627\u0639\u062a \u0645\u0648\u0646\u062f\u0647' + (p.n ? '\u060c ' + p.n : '') + '\n'; });
            if (D.rv) {
              var v = D.rv;
              s += '\u0628\u0627\u0632\u0628\u06cc\u0646\u06cc: \u0627\u0645\u062a\u06cc\u0627\u0632 \u0631\u0648\u0632 ' + (v.sc || '?') + '/5' + ((v.ob || []).length ? ' | \u0645\u0648\u0627\u0646\u0639: ' + v.ob.join('\u060c ') : '') + '\n';
              if (v.res) s += '  \u0645\u0647\u0645\u200c\u062a\u0631\u06cc\u0646 \u0646\u062a\u06cc\u062c\u0647: ' + v.res + '\n';
              if (v.obt) s += '  \u062a\u0648\u0636\u06cc\u062d \u0645\u0627\u0646\u0639: ' + v.obt + '\n';
              if (v.hyp) s += '  \u0641\u0631\u0636\u06cc\u0647 \u0627\u0645\u0631\u0648\u0632: ' + v.hyp + '\n';
              if (v.drop) s += '  \u062d\u0630\u0641 \u06cc\u0627 \u0633\u0627\u062f\u0647\u200c\u0633\u0627\u0632\u06cc: ' + v.drop + '\n';
              if (v.next) s += '  \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc: ' + v.next + '\n';
            }
            return s;
          }
          function weekSummary() {
            var r = stats(lastKeys(7)), s = '### \u062e\u0644\u0627\u0635\u0647 \u06f7 \u0631\u0648\u0632 \u0627\u062e\u06cc\u0631\n';
            s += '\u0631\u0648\u0632\u0647\u0627\u06cc \u062f\u0627\u0631\u0627\u06cc \u062f\u0627\u062f\u0647: ' + r.n + ' \u0627\u0632 \u06f7\n';
            if (r.en) s += '\u0645\u06cc\u0627\u0646\u06af\u06cc\u0646 \u0627\u0646\u0631\u0698\u06cc: ' + r1(r.e / r.en) + '/5\n';
            if (r.sn) s += '\u0645\u06cc\u0627\u0646\u06af\u06cc\u0646 \u062e\u0648\u0627\u0628: ' + r1(r.sl / r.sn) + ' \u0633\u0627\u0639\u062a\n';
            if (r.mainT) s += '\u0627\u0646\u062c\u0627\u0645 \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc: ' + r.mainD + ' \u0627\u0632 ' + r.mainT + '\n';
            if (r.aT) s += '\u0627\u0646\u062c\u0627\u0645 \u06a9\u0627\u0631\u0647\u0627\u06cc \u0636\u0631\u0648\u0631\u06cc (A): ' + r.aD + ' \u0627\u0632 ' + r.aT + '\n';
            if (r.tT) s += '\u06a9\u0644 \u06a9\u0627\u0631\u0647\u0627: ' + r.tD + ' \u0627\u0632 ' + r.tT + '\n';
            var ob = Object.keys(r.ob); if (ob.length) s += '\u0645\u0648\u0627\u0646\u0639: ' + ob.map(function (o) { return o + ' \u00d7' + r.ob[o]; }).join('\u060c ') + '\n';
            if (r.hiS.length >= 2 && r.loS.length >= 2) s += '\u0627\u0646\u0631\u0698\u06cc \u0628\u0639\u062f \u0627\u0632 \u062e\u0648\u0627\u0628 \u06f7+ \u0633\u0627\u0639\u062a: ' + r1(avg(r.hiS)) + ' (' + r.hiS.length + ' \u0631\u0648\u0632) | \u06a9\u0645\u062a\u0631 \u0627\u0632 \u06f7: ' + r1(avg(r.loS)) + ' (' + r.loS.length + ' \u0631\u0648\u0632)\n';
            return s;
          }
          function buildPrompt(mode) {
            var k = dk(), s = (S.set.sp || DEF_PROMPT) + '\n\n---\n' + MODES[mode][1] + '\n\n';
            if (mode == 'week') { s += weekSummary() + '\n'; lastKeys(7).reverse().forEach(function (x) { if (peek(x)) s += dayText(x, false) + '\n'; }); }
            else {
              s += dayText(k, true) + '\n';
              var y = peek(addD(k, -1)); if (y) s += '(\u062f\u06cc\u0631\u0648\u0632 \u0628\u0631\u0627\u06cc \u0645\u0642\u0627\u06cc\u0633\u0647)\n' + dayText(addD(k, -1), false) + '\n';
              if (mode != 'night') s += weekSummary();
            }
            return s;
          }

          /* ---------- UI ---------- */
          var TAB = 'today', EDIT = null, FORM = false, RP = false;
          function go(t) { TAB = t; EDIT = null; FORM = false; RP = false; render(); window.scrollTo(0, 0); }
          function appBack() { if (FORM || EDIT || RP) { FORM = false; EDIT = null; RP = false; render(); return true; } if (TAB != 'today') { go('today'); return true; } return false; }
          function nav() {
            $('nav').innerHTML = TABS.map(function (t) { return '<div class="' + (TAB == t[0] ? 'on' : '') + '" onclick="go(\'' + t[0] + '\')"><span>' + t[1] + '</span>' + t[2] + '</div>'; }).join('');
          }
          function chips(name, n, cur, fn) {
            var h = '<div class="chips">';
            for (var i = 1; i <= n; i++) h += '<span id="' + fn + i + '" class="chip' + (cur == i ? ' on' : '') + '" onclick="' + fn + '(' + i + ')">' + fa(i) + (name ? ' ' + name[i] : '') + '</span>';
            return h + '</div>';
          }
          function render() {
            nav();
            $('dt').textContent = jStr(dk(), true);
            $('ttl').textContent = { today: '\u2600\ufe0f \u0627\u0645\u0631\u0648\u0632', night: '\ud83c\udf19 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0634\u0628', trend: '\ud83d\udcca \u0631\u0648\u0646\u062f', ai: '\ud83e\udde0 \u0647\u0648\u0634 \u0645\u0635\u0646\u0648\u0639\u06cc', set: '\u2699\ufe0f \u062a\u0646\u0638\u06cc\u0645\u0627\u062a' }[TAB];
            var f = { today: rToday, night: rNight, trend: rTrend, ai: rAi, set: rSet }[TAB];
            $('pg').innerHTML = f();
          }

          /* today */
          function rToday() {
            var k = dk(), D = day(k), c = cap(D), h = '';
            if (!D.tasks.length) {
              var cl = carryList(k);
              if (cl.list.length) h += '<div class="box warn"><b>' + fa(cl.list.length) + ' \u06a9\u0627\u0631 \u0646\u0627\u062a\u0645\u0648\u0645 \u0627\u0632 ' + jStr(cl.k) + '</b><div class="mut">\u06a9\u0627\u0631\u0647\u0627\u06cc D \u0645\u0646\u062a\u0642\u0644 \u0646\u0645\u06cc\u0634\u0646. \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc \u0642\u0628\u0644\u06cc\u060c \u062b\u0627\u0646\u0648\u06cc\u0647 \u0645\u06cc\u0634\u0647 \u062a\u0627 \u0627\u0645\u0631\u0648\u0632 \u062e\u0648\u062f\u062a \u0627\u0646\u062a\u062e\u0627\u0628 \u06a9\u0646\u06cc.</div><button class="btn s" onclick="carry()">\u2b05\ufe0f \u0627\u0646\u062a\u0642\u0627\u0644 \u0628\u0647 \u0627\u0645\u0631\u0648\u0632</button> <button class="btn g s" onclick="skipCarry()">\u0646\u0647\u060c \u0627\u0632 \u0635\u0641\u0631</button></div>';
            }
            h += '<div class="box"><h3>\u0648\u0636\u0639\u06cc\u062a \u0627\u0645\u0631\u0648\u0632</h3><label>\u0627\u0646\u0631\u0698\u06cc \u0627\u0644\u0627\u0646</label>' + chips(EN, 5, D.e, 'setE') +
              '<div class="row"><div style="flex:1"><label>\u062e\u0648\u0627\u0628 \u062f\u06cc\u0634\u0628 (\u0633\u0627\u0639\u062a)</label><input inputmode="decimal" value="' + esc(D.sl) + '" onchange="setF(\'sl\',this.value)"></div>' +
              '<div style="flex:1"><label>\u0632\u0645\u0627\u0646 \u0622\u0632\u0627\u062f \u0627\u0645\u0631\u0648\u0632 (\u0633\u0627\u0639\u062a\u060c \u0628\u062f\u0648\u0646 \u0645\u063a\u0627\u0632\u0647)</label><input inputmode="decimal" value="' + esc(D.h) + '" onchange="setF(\'h\',this.value)"></div></div>' +
              '<label>\u062a\u0639\u0647\u062f \u0642\u0637\u0639\u06cc \u0627\u0645\u0631\u0648\u0632 (\u0645\u062b\u0644\u0627\u064b \u0645\u063a\u0627\u0632\u0647 \u06f9 \u062a\u0627 \u06f2\u06f1)</label><input value="' + esc(D.c) + '" onchange="setF(\'c\',this.value)"></div>';
            return h + '<div id="dyn">' + rDyn() + '</div>';
          }
          function rDyn() {
            var k = dk(), D = day(k), c = cap(D), h = '';
            if (c.avail) {
              var p = c.avail ? Math.min(100, Math.round(c.plan / c.avail * 100)) : 0, cls = c.over > 0 ? 'ov' : c.plan > c.lim * 0.85 ? 'hi' : '';
              h += '<div class="box"><div class="row"><b>\u0638\u0631\u0641\u06cc\u062a</b><span class="mut">' + fa(c.plan) + ' \u0627\u0632 ' + fa(c.avail) + ' \u062f\u0642\u06cc\u0642\u0647 \u00b7 \u0633\u0642\u0641 \u0627\u0645\u0646 ' + fa(c.lim) + '</span></div><div class="bar"><i class="' + cls + '" style="width:' + p + '%"></i></div>' +
                '<div class="mut sm">\u0627\u0646\u062c\u0627\u0645\u200c\u0634\u062f\u0647: ' + fa(c.done) + ' \u062f\u0642\u06cc\u0642\u0647 \u00b7 \u0645\u0648\u0646\u062f\u0647: ' + fa(c.left) + ' \u062f\u0642\u06cc\u0642\u0647</div></div>';
            }
            var adv = advice(D);
            if (adv.length) {
              h += '<div class="box' + (adv[0][0] == 'l' ? ' low' : adv[0][0] == 'o' ? ' ok' : ' warn') + '"><h3>\ud83e\udded \u067e\u06cc\u0634\u0646\u0647\u0627\u062f \u0633\u06cc\u0633\u062a\u0645</h3>';
              adv.forEach(function (a) { h += '<div class="adv">\u2022 ' + a[1] + (a[2] == 'trim' ? ' <button class="btn s" onclick="doTrim()">\u0627\u0646\u062a\u0642\u0627\u0644 \u062e\u0648\u062f\u06a9\u0627\u0631</button>' : '') + '</div>'; });
              h += '</div>';
            }
            h += '<div class="box"><div class="row"><h3>\u06a9\u0627\u0631\u0647\u0627\u06cc \u0627\u0645\u0631\u0648\u0632</h3><span><button class="btn s" onclick="openForm()">\u2795 \u06a9\u0627\u0631 \u062c\u062f\u06cc\u062f</button> <button class="btn g s" onclick="RP=!RP;FORM=false;EDIT=null;render()">\ud83d\udd04 \u062a\u0646\u0638\u06cc\u0645 \u0645\u062c\u062f\u062f</button></span></div>';
            if (RP) h += rReplan(D);
            if (FORM && !EDIT) h += rForm(null);
            var ord = D.tasks.slice().sort(function (a, b) {
              var ra = a.r == 'main' ? 0 : a.r == 'sec' ? 1 : 2, rb = b.r == 'main' ? 0 : b.r == 'sec' ? 1 : 2;
              return ((a.d ? 1 : 0) - (b.d ? 1 : 0)) || (ra - rb) || (GORD[b.g] - GORD[a.g]);
            });
            if (!ord.length && !FORM) h += '<div class="mut">\u0647\u0646\u0648\u0632 \u06a9\u0627\u0631\u06cc \u0646\u06cc\u0633\u062a. \u0628\u0627 \u2795 \u0627\u0636\u0627\u0641\u0647 \u06a9\u0646. \u0646\u06a9\u062a\u0647: \u00ab\u0631\u0648\u06cc \u06a9\u0633\u0628\u200c\u0648\u06a9\u0627\u0631 \u06a9\u0627\u0631 \u06a9\u0646\u00bb \u0646\u0646\u0648\u06cc\u0633\u061b \u0628\u0646\u0648\u06cc\u0633 \u00ab\u0641\u0631\u0648\u0634 \u06f7 \u0631\u0648\u0632 \u0627\u062e\u06cc\u0631 \u0631\u0648 \u062b\u0628\u062a \u06a9\u0646 \u0648 \u06f3 \u06a9\u0627\u0644\u0627\u06cc \u06a9\u0645\u200c\u06af\u0631\u062f\u0634 \u0631\u0648 \u067e\u06cc\u062f\u0627 \u06a9\u0646\u00bb.</div>';
            ord.forEach(function (t) { h += EDIT == t.id ? rForm(t) : rTask(t); });
            h += '</div>';
            var done = D.tasks.filter(function (t) { return t.d; }).length;
            if (D.tasks.length) h += '<div class="box"><div class="row"><span>' + fa(done) + ' \u0627\u0632 ' + fa(D.tasks.length) + ' \u0627\u0646\u062c\u0627\u0645 \u0634\u062f</span><button class="btn g s" onclick="go(\'night\')">\ud83c\udf19 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0634\u0628</button></div></div>';
            return h;
          }
          function rTask(t) {
            return '<div class="task' + (t.d ? ' done' : '') + '"><div class="row" style="flex-wrap:nowrap"><span class="ck' + (t.d ? ' on' : '') + '" onclick="tgl(\'' + t.id + '\')">' + (t.d ? '\u2713' : '') + '</span>' +
              '<div class="tt" onclick="EDIT=\'' + t.id + '\';FORM=false;render()"><b>' + (t.r == 'main' ? '\u2b50 ' : t.r == 'sec' ? '\u2606 ' : '') + esc(t.t) + '</b><br><span class="g g' + t.g + '">' + t.g + '</span><span class="mut sm">' + (num(t.m) ? fa(num(t.m)) + ' \u062f\u0642\u06cc\u0642\u0647' : '\u0628\u062f\u0648\u0646 \u0632\u0645\u0627\u0646') + (t.f ? ' \u00b7 \u0627\u0632 ' + jStr(t.f) : '') + '</span>' +
              (t.na ? '<div class="mut sm">\u25b6\ufe0f ' + esc(t.na) + '</div>' : '') + (t.end ? '<div class="mut sm">\ud83c\udfc1 ' + esc(t.end) + '</div>' : '') + '</div></div>' +
              (t.d ? '' : '<div style="margin-top:4px"><button class="btn g s" onclick="setMain(\'' + t.id + '\')">\u2b50 \u0627\u0635\u0644\u06cc</button> <button class="btn g s" onclick="setSec(\'' + t.id + '\')">\u2606 \u062b\u0627\u0646\u0648\u06cc\u0647</button> <button class="btn g s" onclick="tmrw(\'' + t.id + '\')">\u27a1\ufe0f \u0641\u0631\u062f\u0627</button></div>') + '</div>';
          }
          function rForm(t) {
            var x = t || { t: '', g: 'B', m: '', na: '', end: '', r: '' }, g = '';
            ['A', 'B', 'C', 'D'].forEach(function (q) { g += '<span class="chip' + (x.g == q ? ' on' : '') + '" onclick="fG(\'' + q + '\')" id="fg' + q + '">' + q + ' \u00b7 ' + GR[q] + '</span>'; });
            return '<div class="box" style="margin:8px 0;background:#0F2138"><label>\u0639\u0646\u0648\u0627\u0646 \u06a9\u0627\u0631</label><input id="fT" value="' + esc(x.t) + '">' +
              '<label>\u06af\u0631\u0648\u0647</label><div class="chips">' + g + '</div><input type="hidden" id="fGv" value="' + x.g + '">' +
              '<label>\u0632\u0645\u0627\u0646 \u062a\u0642\u0631\u06cc\u0628\u06cc (\u062f\u0642\u06cc\u0642\u0647)</label><input id="fM" inputmode="numeric" value="' + esc(x.m) + '">' +
              '<label>\u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc (\u0627\u0648\u0644\u06cc\u0646 \u062d\u0631\u06a9\u062a \u0641\u06cc\u0632\u06cc\u06a9\u06cc)</label><input id="fN" value="' + esc(x.na) + '">' +
              '<label>\u0645\u0639\u06cc\u0627\u0631 \u067e\u0627\u06cc\u0627\u0646 (\u06a9\u0650\u06cc \u062a\u0645\u0648\u0645\u0647\u061f)</label><input id="fE" value="' + esc(x.end) + '">' +
              '<button class="btn" onclick="saveTask(' + (t ? '\'' + t.id + '\'' : 'null') + ')">\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647</button> <button class="btn g" onclick="FORM=false;EDIT=null;render()">\u0627\u0646\u0635\u0631\u0627\u0641</button>' +
              (t ? ' <button class="btn r" onclick="delTask(\'' + t.id + '\')">\ud83d\uddd1 \u062d\u0630\u0641</button>' + (t.d ? '' : ' <button class="btn g" onclick="setNone(\'' + t.id + '\')">\u0639\u0627\u062f\u06cc \u06a9\u0646</button>') : '') + '</div>';
          }
          function rReplan(D) {
            return '<div class="box" style="margin:8px 0;background:#0F2138"><b>\ud83d\udd04 \u0634\u0631\u0627\u06cc\u0637 \u0639\u0648\u0636 \u0634\u062f\u061f</b><div class="mut sm">\u0645\u062b\u0644\u0627\u064b \u0645\u063a\u0627\u0632\u0647 \u0634\u0644\u0648\u063a \u0634\u062f \u06cc\u0627 \u06a9\u0627\u0631\u06cc \u0628\u06cc\u0634\u062a\u0631 \u0637\u0648\u0644 \u06a9\u0634\u06cc\u062f. \u0628\u0631\u0646\u0627\u0645\u0647 \u0631\u0648 \u0628\u0627 \u0638\u0631\u0641\u06cc\u062a \u062c\u062f\u06cc\u062f \u0645\u06cc\u200c\u0686\u06cc\u0646\u06cc\u0645\u060c \u0646\u0647 \u0628\u0647 \u0647\u0631 \u0642\u06cc\u0645\u062a\u06cc.</div>' +
              '<label>\u0627\u0646\u0631\u0698\u06cc \u0627\u0644\u0627\u0646</label>' + chips(EN, 5, 0, 'rpE') + '<input type="hidden" id="rpEv" value="' + (D.e || 0) + '">' +
              '<label>\u0686\u0646\u062f \u0633\u0627\u0639\u062a \u0622\u0632\u0627\u062f \u062a\u0627 \u0622\u062e\u0631 \u0627\u0645\u0631\u0648\u0632 \u0645\u0648\u0646\u062f\u0647\u061f</label><input id="rpH" inputmode="decimal">' +
              '<label>\u0686\u06cc \u0634\u062f\u061f (\u0627\u062e\u062a\u06cc\u0627\u0631\u06cc)</label><input id="rpN"><button class="btn" onclick="doReplan()">\u2705 \u062f\u0648\u0628\u0627\u0631\u0647 \u0628\u0686\u06cc\u0646</button></div>';
          }
          function setE(n) { day(dk()).e = n; save(); for (var i = 1; i <= 5; i++) { var el = $('setE' + i); if (el) el.className = 'chip' + (i == n ? ' on' : ''); } var d = $('dyn'); if (d) d.innerHTML = rDyn(); }
          function setF(f, v) { day(dk())[f] = v; save(); var el = $('dyn'); if (el) el.innerHTML = rDyn(); }
          function openForm() { FORM = true; EDIT = null; render(); setTimeout(function () { var el = $('fT'); if (el) el.focus(); }, 50); }
          function fG(q) { $('fGv').value = q; ['A', 'B', 'C', 'D'].forEach(function (z) { $('fg' + z).className = 'chip' + (z == q ? ' on' : ''); }); }
          function saveTask(id) {
            var D = day(dk()), title = $('fT').value.replace(/^\s+|\s+$/g, '');
            if (!title) { toast('\u0639\u0646\u0648\u0627\u0646 \u0631\u0648 \u0628\u0646\u0648\u06cc\u0633'); return; }
            var t = id ? findT(D, id) : null;
            if (!t) { t = { id: uid(), r: '', d: false }; D.tasks.push(t); }
            t.t = title; t.g = $('fGv').value; t.m = String(num($('fM').value) || ''); t.na = $('fN').value; t.end = $('fE').value;
            FORM = false; EDIT = null; save(); render();
          }
          function delTask(id) { if (!confirm('\u0627\u06cc\u0646 \u06a9\u0627\u0631 \u062d\u0630\u0641 \u0628\u0634\u0647\u061f')) return; var D = day(dk()); D.tasks = D.tasks.filter(function (t) { return t.id != id; }); EDIT = null; save(); render(); }
          function tgl(id) { var t = findT(day(dk()), id); if (!t) return; t.d = !t.d; t.dt = t.d ? Date.now() : 0; save(); render(); if (t.d && t.r == 'main') toast('\u2b50 \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc \u0627\u0645\u0631\u0648\u0632 \u0627\u0646\u062c\u0627\u0645 \u0634\u062f'); }
          function setMain(id) { var D = day(dk()); D.tasks.forEach(function (t) { if (t.r == 'main') t.r = 'sec'; }); var t = findT(D, id); t.r = 'main'; save(); render(); }
          function setSec(id) {
            var D = day(dk()), t = findT(D, id), n = D.tasks.filter(function (x) { return x.r == 'sec' && !x.d; }).length;
            if (t.r != 'sec' && n >= 2) { toast('\u062d\u062f\u0627\u06a9\u062b\u0631 \u06f2 \u0627\u0648\u0644\u0648\u06cc\u062a \u062b\u0627\u0646\u0648\u06cc\u0647. \u06cc\u06a9\u06cc \u0631\u0648 \u0639\u0627\u062f\u06cc \u06a9\u0646'); return; }
            t.r = t.r == 'sec' ? '' : 'sec'; save(); render();
          }
          function setNone(id) { var t = findT(day(dk()), id); t.r = ''; EDIT = null; save(); render(); }
          function tmrw(id) { var k = dk(), D = day(k), t = findT(D, id); moveTask(k, t, addD(k, 1)); save(); render(); toast('\u27a1\ufe0f \u0631\u0641\u062a \u0628\u0631\u0627\u06cc \u0641\u0631\u062f\u0627'); }
          function doTrim() {
            var k = dk(), D = day(k), list = trimPlan(D);
            if (!list.length) { toast('\u0686\u06cc\u0632\u06cc \u0628\u0631\u0627\u06cc \u0627\u0646\u062a\u0642\u0627\u0644 \u0646\u06cc\u0633\u062a\u061b \u06a9\u0627\u0631\u0647\u0627\u06cc A \u0648 \u0627\u0635\u0644\u06cc \u062f\u0633\u062a \u0646\u0645\u06cc\u200c\u062e\u0648\u0631\u0646'); return; }
            if (!confirm(fa(list.length) + ' \u06a9\u0627\u0631 \u0628\u0631\u0647 \u0628\u0631\u0627\u06cc \u0641\u0631\u062f\u0627\u061f\n\n' + list.map(function (t) { return '\u2022 ' + t.g + ' ' + t.t; }).join('\n'))) return;
            list.forEach(function (t) { moveTask(k, t, addD(k, 1)); });
            save(); render(); toast('\u2705 \u0628\u0631\u0646\u0627\u0645\u0647 \u0633\u0628\u06a9 \u0634\u062f');
          }
          function rpE(n) { $('rpEv').value = n; for (var i = 1; i <= 5; i++) { var el = $('rpE' + i); if (el) el.className = 'chip' + (i == n ? ' on' : ''); } }
          function doReplan() {
            var D = day(dk()), e = +$('rpEv').value, hh = $('rpH').value, d = new Date();
            if (!num(hh)) { toast('\u0633\u0627\u0639\u062a\u200c\u0647\u0627\u06cc \u0645\u0648\u0646\u062f\u0647 \u0631\u0648 \u0628\u0646\u0648\u06cc\u0633'); return; }
            var doneM = cap(D).done;
            D.rp.push({ tm: pad(d.getHours()) + ':' + pad(d.getMinutes()), e: e, h: num(hh), n: $('rpN').value });
            if (e) D.e = e;
            D.h = String(r1(num(hh) + doneM / 60));
            RP = false; save(); render();
            if (trimPlan(D).length) setTimeout(doTrim, 200); else toast('\u2705 \u0628\u0627 \u0638\u0631\u0641\u06cc\u062a \u062c\u062f\u06cc\u062f \u062c\u0648\u0631 \u062f\u0631\u0645\u06cc\u0627\u062f');
          }
          function carry() {
            var k = dk(), cl = carryList(k);
            cl.list.forEach(function (t) { moveTask(cl.k, t, k); });
            save(); render(); toast('\u2705 ' + fa(cl.list.length) + ' \u06a9\u0627\u0631 \u0645\u0646\u062a\u0642\u0644 \u0634\u062f');
          }
          function skipCarry() { var cl = carryList(dk()); cl.list.forEach(function (t) { t.mv = 'skip'; }); save(); render(); }

          /* night */
          var RV = null;
          function rNight() {
            var k = dk(), D = day(k), v = RV || (D.rv ? JSON.parse(JSON.stringify(D.rv)) : { ob: [], sc: 0, res: '', obt: '', hyp: '', drop: '', next: '' });
            RV = v;
            var done = D.tasks.filter(function (t) { return t.d; }), open = D.tasks.filter(function (t) { return !t.d && !t.mv; });
            var main = D.tasks.filter(function (t) { return t.r == 'main'; })[0];
            var h = '<div class="box"><h3>\u0686\u06cc \u0648\u0627\u0642\u0639\u0627\u064b \u0627\u0646\u062c\u0627\u0645 \u0634\u062f\u061f</h3>';
            if (!D.tasks.length) h += '<div class="mut">\u0627\u0645\u0631\u0648\u0632 \u06a9\u0627\u0631\u06cc \u062b\u0628\u062a \u0646\u0634\u062f\u0647.</div>';
            else {
              h += '<div class="big">' + fa(done.length) + ' \u0627\u0632 ' + fa(D.tasks.length) + '</div>';
              if (main) h += '<div>' + (main.d ? '\u2705' : '\u274c') + ' \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc: ' + esc(main.t) + '</div>';
              if (open.length) h += '<div class="mut sm">\u0646\u0627\u062a\u0645\u0648\u0645: ' + open.map(function (t) { return esc(t.t); }).join('\u060c ') + '</div>';
            }
            h += '<div class="mut sm" style="margin-top:6px">\u0627\u06cc\u0646 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0633\u06cc\u0633\u062a\u0645 \u0631\u0648 \u0645\u06cc\u200c\u0633\u0646\u062c\u0647\u060c \u0646\u0647 \u062a\u0648 \u0631\u0648.</div></div>';
            h += '<div class="box"><label>\u0627\u0645\u062a\u06cc\u0627\u0632 \u0627\u0645\u0631\u0648\u0632 (\u067e\u06cc\u0634\u0631\u0641\u062a \u0648\u0627\u0642\u0639\u06cc\u060c \u0646\u0647 \u062d\u062c\u0645 \u06a9\u0627\u0631)</label>' + chips(null, 5, v.sc, 'rvS') +
              '<label>\u0645\u0647\u0645\u200c\u062a\u0631\u06cc\u0646 \u0646\u062a\u06cc\u062c\u0647 \u0627\u0645\u0631\u0648\u0632</label><input id="rvRes" value="' + esc(v.res) + '">' +
              '<label>\u0686\u06cc \u0645\u0627\u0646\u0639 \u0634\u062f\u061f (\u0686\u0646\u062f\u062a\u0627 \u0645\u06cc\u0634\u0647)</label><div class="chips">' + OBS.map(function (o, i) { return '<span class="chip' + (v.ob.indexOf(o) >= 0 ? ' on' : '') + '" onclick="rvO(' + i + ')">' + o + '</span>'; }).join('') + '</div>' +
              '<label>\u062a\u0648\u0636\u06cc\u062d \u0645\u0627\u0646\u0639 (\u0627\u062e\u062a\u06cc\u0627\u0631\u06cc)</label><input id="rvObt" value="' + esc(v.obt) + '">' +
              '<label>\u06a9\u062f\u0648\u0645 \u0641\u0631\u0636\u06cc\u0647 \u0627\u0645\u0631\u0648\u0632 \u062a\u0623\u06cc\u06cc\u062f \u06cc\u0627 \u0631\u062f \u0634\u062f\u061f (\u0627\u062e\u062a\u06cc\u0627\u0631\u06cc)</label><input id="rvHyp" value="' + esc(v.hyp) + '" placeholder="\u0645\u062b\u0644\u0627\u064b: \u0628\u0639\u062f \u0627\u0632 \u0645\u063a\u0627\u0632\u0647 \u06a9\u0627\u0631 \u0641\u06a9\u0631\u06cc \u0646\u0645\u06cc\u200c\u062a\u0648\u0646\u0645">' +
              '<label>\u0686\u06cc \u0628\u0627\u06cc\u062f \u062d\u0630\u0641 \u06cc\u0627 \u0633\u0627\u062f\u0647 \u0628\u0634\u0647\u061f</label><input id="rvDrop" value="' + esc(v.drop) + '">' +
              '<label>\u0645\u0647\u0645\u200c\u062a\u0631\u06cc\u0646 \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc (\u062e\u0648\u062f\u06a9\u0627\u0631 \u0645\u06cc\u0631\u0647 \u062a\u0648 \u06a9\u0627\u0631\u0647\u0627\u06cc \u0641\u0631\u062f\u0627)</label><input id="rvNext" value="' + esc(v.next) + '">' +
              '<button class="btn w" onclick="saveRv()">\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc</button>' + (D.rv ? '<div class="mut sm">\u0630\u062e\u06cc\u0631\u0647 \u0634\u062f\u0647 \u2705 (\u062f\u0648\u0628\u0627\u0631\u0647 \u0630\u062e\u06cc\u0631\u0647 \u06a9\u0646\u06cc\u060c \u0628\u0647\u200c\u0631\u0648\u0632 \u0645\u06cc\u0634\u0647)</div>' : '') + '</div>';
            if (open.length) h += '<div class="box"><div class="row"><span>' + fa(open.length) + ' \u06a9\u0627\u0631 \u0646\u0627\u062a\u0645\u0648\u0645</span><button class="btn g s" onclick="openTmrw()">\u27a1\ufe0f \u0647\u0645\u0647 \u0628\u0631\u0646 \u0641\u0631\u062f\u0627</button></div><div class="mut sm">\u06a9\u0627\u0631\u0647\u0627\u06cc D \u0645\u0646\u062a\u0642\u0644 \u0646\u0645\u06cc\u0634\u0646.</div></div>';
            return h;
          }
          function rvGrab() { if (!$('rvRes')) return; RV.res = $('rvRes').value; RV.obt = $('rvObt').value; RV.hyp = $('rvHyp').value; RV.drop = $('rvDrop').value; RV.next = $('rvNext').value; }
          function rvS(n) { rvGrab(); RV.sc = n; render(); }
          function rvO(i) { rvGrab(); var o = OBS[i], p = RV.ob.indexOf(o); if (p >= 0) RV.ob.splice(p, 1); else RV.ob.push(o); render(); }
          function saveRv() {
            rvGrab();
            var k = dk(), D = day(k), old = D.rv;
            if (!RV.sc) { toast('\u0627\u0645\u062a\u06cc\u0627\u0632 \u0631\u0648\u0632 \u0631\u0648 \u0628\u0632\u0646'); return; }
            RV.t = Date.now();
            D.rv = JSON.parse(JSON.stringify(RV));
            var nx = String(RV.next || '').replace(/^\s+|\s+$/g, ''), T = day(addD(k, 1));
            if (old && old.nid) T.tasks = T.tasks.filter(function (t) { return t.id != old.nid; });
            if (nx) { var n = { id: uid(), t: nx, g: 'B', m: '', na: '', end: '', r: 'sec', d: false, f: k }; T.tasks.push(n); D.rv.nid = n.id; }
            save(); render(); toast('\u2705 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0630\u062e\u06cc\u0631\u0647 \u0634\u062f' + (nx ? '\u060c \u0627\u0642\u062f\u0627\u0645 \u0628\u0639\u062f\u06cc \u0631\u0641\u062a \u0628\u0631\u0627\u06cc \u0641\u0631\u062f\u0627' : ''));
          }
          function openTmrw() {
            var k = dk(), D = day(k), l = D.tasks.filter(function (t) { return !t.d && !t.mv && t.g != 'D'; });
            l.forEach(function (t) { moveTask(k, t, addD(k, 1)); });
            save(); render(); toast('\u2705 ' + fa(l.length) + ' \u06a9\u0627\u0631 \u0631\u0641\u062a \u0628\u0631\u0627\u06cc \u0641\u0631\u062f\u0627');
          }

          /* trend */
          function rTrend() {
            var keys = lastKeys(7), r = stats(keys), h = '';
            if (r.n < 5) h += '<div class="box warn">\u0641\u0642\u0637 ' + fa(r.n) + ' \u0631\u0648\u0632 \u062f\u0627\u062f\u0647 \u062f\u0627\u0631\u06cc. \u062a\u0627 \u06f5 \u0631\u0648\u0632 \u0646\u0634\u062f\u0647\u060c \u0627\u06cc\u0646\u200c\u0647\u0627 \u0631\u0648 \u0627\u0644\u06af\u0648 \u062d\u0633\u0627\u0628 \u0646\u06a9\u0646\u061b \u0641\u0642\u0637 \u062b\u0628\u062a \u06a9\u0646.</div>';
            h += '<div class="box"><h3>\u06f7 \u0631\u0648\u0632 \u0627\u062e\u06cc\u0631</h3><div class="row">' +
              cell('\u0627\u0646\u0631\u0698\u06cc', r.en ? fa(r1(r.e / r.en)) + '/\u06f5' : '\u2014') + cell('\u062e\u0648\u0627\u0628', r.sn ? fa(r1(r.sl / r.sn)) + ' \u0633' : '\u2014') +
              cell('\u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc', r.mainT ? fa(r.mainD) + '/' + fa(r.mainT) : '\u2014') + cell('\u06a9\u0627\u0631\u0647\u0627\u06cc A', r.aT ? fa(Math.round(r.aD / r.aT * 100)) + '\u066a' : '\u2014') + '</div></div>';
            if (r.hiS.length >= 2 && r.loS.length >= 2) {
              var a = avg(r.hiS), b = avg(r.loS);
              h += '<div class="box"><h3>\ud83e\uddea \u062e\u0648\u0627\u0628 \u0648 \u0627\u0646\u0631\u0698\u06cc (\u062f\u0627\u062f\u0647 \u062e\u0648\u062f\u062a)</h3><div>\u062e\u0648\u0627\u0628 \u06f7+ \u0633\u0627\u0639\u062a: \u0627\u0646\u0631\u0698\u06cc ' + fa(r1(a)) + ' (' + fa(r.hiS.length) + ' \u0631\u0648\u0632)</div><div>\u062e\u0648\u0627\u0628 \u06a9\u0645\u062a\u0631 \u0627\u0632 \u06f7: \u0627\u0646\u0631\u0698\u06cc ' + fa(r1(b)) + ' (' + fa(r.loS.length) + ' \u0631\u0648\u0632)</div><div class="mut sm">' +
                (Math.abs(a - b) >= 0.7 ? '\u062a\u0641\u0627\u0648\u062a \u0642\u0627\u0628\u0644\u200c\u062a\u0648\u062c\u0647\u0647\u060c \u0648\u0644\u06cc \u0628\u0627 \u0686\u0646\u062f \u0631\u0648\u0632 \u0647\u0646\u0648\u0632 \u0641\u0631\u0636\u06cc\u0647\u200c\u0633\u062a \u0646\u0647 \u0648\u0627\u0642\u0639\u06cc\u062a.' : '\u0641\u0639\u0644\u0627\u064b \u062a\u0641\u0627\u0648\u062a \u0645\u0639\u0646\u06cc\u200c\u062f\u0627\u0631\u06cc \u062f\u06cc\u062f\u0647 \u0646\u0645\u06cc\u0634\u0647.') + '</div></div>';
            }
            var ob = Object.keys(r.ob).sort(function (x, y) { return r.ob[y] - r.ob[x]; });
            if (ob.length) {
              var mx = r.ob[ob[0]];
              h += '<div class="box"><h3>\u0645\u0648\u0627\u0646\u0639 \u067e\u0631\u062a\u06a9\u0631\u0627\u0631</h3>' + ob.map(function (o) { return '<div class="row"><span>' + o + '</span><span class="mut">' + fa(r.ob[o]) + ' \u0628\u0627\u0631</span></div><div class="bar"><i class="hi" style="width:' + Math.round(r.ob[o] / mx * 100) + '%"></i></div>'; }).join('') + '</div>';
            }
            h += '<div class="box"><h3>\u0631\u0648\u0632\u0647\u0627</h3><table><tr><th>\u0631\u0648\u0632</th><th>\u0627\u0646\u0631\u0698\u06cc</th><th>\u062e\u0648\u0627\u0628</th><th>\u2b50</th><th>\u0627\u0646\u062c\u0627\u0645</th><th>\u0627\u0645\u062a\u06cc\u0627\u0632</th></tr>';
            keys.forEach(function (k) {
              var D = peek(k) || newDay(), m = D.tasks.filter(function (t) { return t.r == 'main'; })[0], dn = D.tasks.filter(function (t) { return t.d; }).length;
              h += '<tr><td>' + jStr(k) + '</td><td>' + (D.e ? fa(D.e) : '\u2014') + '</td><td>' + (num(D.sl) ? fa(num(D.sl)) : '\u2014') + '</td><td>' + (m ? (m.d ? '\u2705' : '\u274c') : '\u2014') + '</td><td>' + (D.tasks.length ? fa(dn) + '/' + fa(D.tasks.length) : '\u2014') + '</td><td>' + (D.rv && D.rv.sc ? fa(D.rv.sc) : '\u2014') + '</td></tr>';
            });
            h += '</table></div>';
            return h;
          }
          function cell(t, v) { return '<div style="flex:1;text-align:center;min-width:70px"><div class="mut sm">' + t + '</div><div class="big">' + v + '</div></div>'; }

          /* ai */
          function rAi() {
            var D = day(dk()), h = '<div class="box"><div class="mut">\u062f\u0627\u062f\u0647\u200c\u0647\u0627 + \u067e\u0631\u0627\u0645\u067e\u062a \u0627\u0635\u0644\u06cc \u06a9\u067e\u06cc \u0645\u06cc\u0634\u0647. \u062a\u0648 \u0686\u062a ClickUp Brain (\u06cc\u0627 \u0647\u0631 \u0647\u0648\u0634 \u0645\u0635\u0646\u0648\u0639\u06cc) \u067e\u06cc\u0633\u062a \u06a9\u0646. \u0628\u062f\u0648\u0646 \u06a9\u0644\u06cc\u062f API \u0648 \u0628\u062f\u0648\u0646 \u0647\u0632\u06cc\u0646\u0647.</div></div><div class="box">';
            Object.keys(MODES).forEach(function (m) { h += '<button class="btn w" onclick="cp(\'' + m + '\')">\ud83d\udccb \u06a9\u067e\u06cc: ' + MODES[m][0] + '</button>'; });
            h += '<button class="btn g w" onclick="preview()">\ud83d\udc41 \u067e\u06cc\u0634\u200c\u0646\u0645\u0627\u06cc\u0634 \u0645\u062a\u0646 \u0635\u0628\u062d</button><div id="pv"></div></div>';
            h += '<div class="box"><label>\u062c\u0648\u0627\u0628 \u0647\u0648\u0634 \u0645\u0635\u0646\u0648\u0639\u06cc \u0631\u0648 \u0627\u06cc\u0646\u062c\u0627 \u0646\u06af\u0647 \u062f\u0627\u0631 (\u0628\u0631\u0627\u06cc \u0627\u0645\u0631\u0648\u0632)</label><textarea id="aiT" style="min-height:120px">' + esc(D.ai) + '</textarea><button class="btn" onclick="saveAi()">\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647</button> <button class="btn g" onclick="pasteAi()">\ud83d\udce5 \u0686\u0633\u0628\u0627\u0646\u062f\u0646</button></div>';
            return h;
          }
          function setClip(t) { try { Android.setClip(t); return true; } catch (e) { try { navigator.clipboard.writeText(t); return true; } catch (e2) { return false; } } }
          function cp(m) { var t = buildPrompt(m); toast(setClip(t) ? '\u2705 \u06a9\u067e\u06cc \u0634\u062f (' + fa(t.length) + ' \u062d\u0631\u0641). \u062d\u0627\u0644\u0627 \u062a\u0648 \u0686\u062a \u067e\u06cc\u0633\u062a \u06a9\u0646' : '\u06a9\u067e\u06cc \u0646\u0634\u062f'); }
          function preview() { $('pv').innerHTML = '<textarea style="min-height:240px;font-size:12px" readonly>' + esc(buildPrompt('morning')) + '</textarea>'; }
          function saveAi() { day(dk()).ai = $('aiT').value; save(); toast('\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647 \u0634\u062f'); }
          function pasteAi() { var t = ''; try { t = Android.clip(); } catch (e) {} if (!t) { toast('\u06a9\u0644\u06cc\u067e\u200c\u0628\u0648\u0631\u062f \u062e\u0627\u0644\u06cc\u0647'); return; } $('aiT').value = t; saveAi(); }

          /* settings */
          function rSet() {
            var s = S.set;
            return '<div class="box"><h3>\ud83d\udd14 \u06cc\u0627\u062f\u0622\u0648\u0631 \u0631\u0648\u0632\u0627\u0646\u0647</h3><label><input type="checkbox" style="width:auto" ' + (s.rem ? 'checked' : '') + ' onchange="S.set.rem=this.checked;save();remind();render()"> \u0631\u0648\u0634\u0646</label>' +
              '<div class="row"><div style="flex:1"><label>\u0635\u0628\u062d</label><input type="time" value="' + s.mt + '" onchange="S.set.mt=this.value;save();remind()"></div><div style="flex:1"><label>\u0634\u0628</label><input type="time" value="' + s.nt + '" onchange="S.set.nt=this.value;save();remind()"></div></div>' +
              '<div class="mut sm">\u0628\u0639\u062f \u0627\u0632 \u0631\u06cc\u0633\u062a\u0627\u0631\u062a \u06af\u0648\u0634\u06cc\u060c \u06cc\u0647 \u0628\u0627\u0631 \u0627\u067e \u0631\u0648 \u0628\u0627\u0632 \u06a9\u0646 \u062a\u0627 \u06cc\u0627\u062f\u0622\u0648\u0631\u0647\u0627 \u062f\u0648\u0628\u0627\u0631\u0647 \u062a\u0646\u0638\u06cc\u0645 \u0628\u0634\u0646.</div><button class="btn g s" onclick="testN()">\ud83e\uddea \u0646\u0648\u062a\u06cc\u0641 \u0622\u0632\u0645\u0627\u06cc\u0634\u06cc (\u06f1\u06f0 \u062b\u0627\u0646\u06cc\u0647 \u0628\u0639\u062f)</button></div>' +
              '<div class="box"><h3>\ud83d\udd53 \u0634\u0631\u0648\u0639 \u0631\u0648\u0632</h3><label>\u062a\u0627 \u0633\u0627\u0639\u062a \u0686\u0646\u062f \u0628\u0627\u0645\u062f\u0627\u062f \u0647\u0646\u0648\u0632 \u00ab\u062f\u06cc\u0631\u0648\u0632\u00bb \u062d\u0633\u0627\u0628 \u0628\u0634\u0647\u061f</label><input inputmode="numeric" value="' + fa(s.ds) + '" onchange="S.set.ds=Math.max(0,Math.min(8,Math.round(num(this.value))));save();render()"><div class="mut sm">\u0645\u062b\u0644\u0627\u064b \u06f4: \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0633\u0627\u0639\u062a \u06f1 \u0634\u0628 \u0647\u0646\u0648\u0632 \u0645\u0627\u0644 \u0627\u0645\u0631\u0648\u0632\u0647.</div></div>' +
              '<div class="box"><h3>\ud83e\udde0 \u067e\u0631\u0627\u0645\u067e\u062a \u0627\u0635\u0644\u06cc</h3><div class="mut sm">\u062e\u0627\u0644\u06cc \u0628\u0630\u0627\u0631\u06cc\u060c \u0646\u0633\u062e\u0647 \u062e\u0644\u0627\u0635\u0647 \u067e\u06cc\u0634\u200c\u0641\u0631\u0636 \u0627\u0633\u062a\u0641\u0627\u062f\u0647 \u0645\u06cc\u0634\u0647. \u0645\u06cc\u200c\u062a\u0648\u0646\u06cc \u067e\u0631\u0627\u0645\u067e\u062a \u06a9\u0627\u0645\u0644\u062a \u0631\u0648 \u0627\u06cc\u0646\u062c\u0627 \u0628\u0630\u0627\u0631\u06cc.</div><textarea id="spT" style="min-height:140px;font-size:12px">' + esc(s.sp || DEF_PROMPT) + '</textarea><button class="btn s" onclick="saveSp()">\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647</button> <button class="btn g s" onclick="S.set.sp=\'\';save();render();toast(\'\u067e\u06cc\u0634\u200c\u0641\u0631\u0636 \u0628\u0631\u06af\u0634\u062a\')">\u21a9\ufe0f \u067e\u06cc\u0634\u200c\u0641\u0631\u0636</button></div>' +
              '<div class="box"><h3>\ud83d\udcbe \u067e\u0634\u062a\u06cc\u0628\u0627\u0646</h3><div class="mut sm">\u06a9\u0644 \u062f\u0627\u062f\u0647\u200c\u0647\u0627 \u06a9\u067e\u06cc \u0645\u06cc\u0634\u0647. \u06cc\u0647 \u062c\u0627\u06cc \u0627\u0645\u0646 \u0646\u06af\u0647\u0634 \u062f\u0627\u0631 (\u0645\u062b\u0644\u0627\u064b \u06cc\u0647 \u062a\u0633\u06a9 ClickUp).</div><button class="btn s" onclick="backup()">\ud83d\udccb \u06a9\u067e\u06cc \u067e\u0634\u062a\u06cc\u0628\u0627\u0646</button> <button class="btn g s" onclick="restore()">\ud83d\udce5 \u0628\u0627\u0632\u06cc\u0627\u0628\u06cc \u0627\u0632 \u06a9\u0644\u06cc\u067e\u200c\u0628\u0648\u0631\u062f</button></div>' +
              '<div class="box"><h3>\ud83c\udd95 \u0622\u067e\u062f\u06cc\u062a</h3><div class="mut">\u0646\u0633\u062e\u0647 \u0641\u0639\u0644\u06cc: v' + fa(BUILD) + '</div><button class="btn s" onclick="checkUpdate(true)">\u0628\u0631\u0631\u0633\u06cc \u0622\u067e\u062f\u06cc\u062a</button></div>' +
              '<div class="box"><button class="btn r s" onclick="resetAll()">\ud83d\uddd1 \u067e\u0627\u06a9 \u06a9\u0631\u062f\u0646 \u0647\u0645\u0647 \u062f\u0627\u062f\u0647\u200c\u0647\u0627</button></div>';
          }
          function saveSp() { var v = $('spT').value; S.set.sp = v == DEF_PROMPT ? '' : v; save(); toast('\ud83d\udcbe \u0630\u062e\u06cc\u0631\u0647 \u0634\u062f'); }
          function backup() { var t = 'LIFE1:' + JSON.stringify(S); toast(setClip(t) ? '\u2705 \u067e\u0634\u062a\u06cc\u0628\u0627\u0646 \u06a9\u067e\u06cc \u0634\u062f (' + fa(Object.keys(S.days).length) + ' \u0631\u0648\u0632)' : '\u06a9\u067e\u06cc \u0646\u0634\u062f'); }
          function restore() {
            var t = ''; try { t = Android.clip(); } catch (e) {}
            t = String(t || '').replace(/^\s+|\s+$/g, '');
            if (t.indexOf('LIFE1:') != 0) { toast('\u0645\u062a\u0646 \u067e\u0634\u062a\u06cc\u0628\u0627\u0646 \u062a\u0648 \u06a9\u0644\u06cc\u067e\u200c\u0628\u0648\u0631\u062f \u0646\u06cc\u0633\u062a'); return; }
            try { var o = JSON.parse(t.substring(6)); if (!o.days) throw 1; if (!confirm('\u062f\u0627\u062f\u0647\u200c\u0647\u0627\u06cc \u0641\u0639\u0644\u06cc \u0628\u0627 \u067e\u0634\u062a\u06cc\u0628\u0627\u0646 (' + fa(Object.keys(o.days).length) + ' \u0631\u0648\u0632) \u062c\u0627\u06cc\u06af\u0632\u06cc\u0646 \u0628\u0634\u0647\u061f')) return; S = o; if (!S.set) S.set = {}; save(); remind(); render(); toast('\u2705 \u0628\u0627\u0632\u06cc\u0627\u0628\u06cc \u0634\u062f'); } catch (e) { toast('\u067e\u0634\u062a\u06cc\u0628\u0627\u0646 \u062e\u0631\u0627\u0628\u0647'); }
          }
          function resetAll() { if (!confirm('\u0647\u0645\u0647 \u0631\u0648\u0632\u0647\u0627 \u0648 \u062a\u0646\u0638\u06cc\u0645\u0627\u062a \u067e\u0627\u06a9 \u0628\u0634\u0647\u061f \u0628\u0631\u06af\u0634\u062a \u0646\u062f\u0627\u0631\u0647.')) return; if (!confirm('\u0645\u0637\u0645\u0626\u0646\u06cc\u061f \u0627\u0648\u0644 \u067e\u0634\u062a\u06cc\u0628\u0627\u0646 \u0628\u06af\u06cc\u0631.')) return; S = { v: 1, days: {}, set: S.set }; save(); go('today'); }

          /* reminders */
          function alarmAt(dayOffset, hm) { var p = String(hm || '08:00').split(':'), d = new Date(); d.setDate(d.getDate() + dayOffset); d.setHours(+p[0] || 0, +p[1] || 0, 0, 0); return d.getTime(); }
          function remind() {
            for (var i = 0; i < 7; i++) {
              try {
                if (!S.set.rem) { Android.cancelAlarm('m' + i); Android.cancelAlarm('n' + i); continue; }
                Android.alarm('m' + i, String(alarmAt(i, S.set.mt)), '\u2600\ufe0f \u0628\u0631\u0646\u0627\u0645\u0647 \u0627\u0645\u0631\u0648\u0632', '\u0627\u0646\u0631\u0698\u06cc\u060c \u0632\u0645\u0627\u0646 \u0622\u0632\u0627\u062f \u0648 \u0627\u0648\u0644\u0648\u06cc\u062a \u0627\u0635\u0644\u06cc \u0631\u0648 \u062b\u0628\u062a \u06a9\u0646 (\u06f1 \u062f\u0642\u06cc\u0642\u0647)');
                Android.alarm('n' + i, String(alarmAt(i, S.set.nt)), '\ud83c\udf19 \u0628\u0627\u0632\u0628\u06cc\u0646\u06cc \u0627\u0645\u0631\u0648\u0632', '\u0686\u06cc \u0627\u0646\u062c\u0627\u0645 \u0634\u062f\u061f \u0686\u06cc \u0645\u0627\u0646\u0639 \u0634\u062f\u061f \u0641\u0631\u062f\u0627 \u0686\u06cc \u0639\u0648\u0636 \u0628\u0634\u0647\u061f');
              } catch (e) {}
            }
          }
          function testN() { try { Android.alarm('test', String(Date.now() + 10000), '\ud83e\uddea \u062a\u0633\u062a', '\u06cc\u0627\u062f\u0622\u0648\u0631\u0647\u0627 \u06a9\u0627\u0631 \u0645\u06cc\u200c\u06a9\u0646\u0646 \u2705'); toast('\u06f1\u06f0 \u062b\u0627\u0646\u06cc\u0647 \u062f\u06cc\u06af\u0647 \u0646\u0648\u062a\u06cc\u0641 \u0645\u06cc\u0627\u062f'); } catch (e) { toast('\u0641\u0642\u0637 \u062a\u0648 \u0627\u067e \u06a9\u0627\u0631 \u0645\u06cc\u200c\u06a9\u0646\u0647'); } }

          /* http + update */
          var CBS = {}, CBN = 0;
          window.__cb = function (id, s) { var f = CBS[id]; delete CBS[id]; var r = {}; try { r = JSON.parse(s); } catch (e) {} if (f) f(r); };
          function http(url, fn) { var id = 'c' + (++CBN); CBS[id] = fn; try { Android.get(url, id); } catch (e) { delete CBS[id]; fn({ status: 0 }); } }
          function checkUpdate(manual) {
            if (!BUILD || REPO.indexOf('/') < 0) { if (manual) toast('\u0622\u067e\u062f\u06cc\u062a \u0641\u0642\u0637 \u062a\u0648 \u0646\u0633\u062e\u0647 \u0633\u0627\u062e\u062a\u0647\u200c\u0634\u062f\u0647 \u06a9\u0627\u0631 \u0645\u06cc\u200c\u06a9\u0646\u0647'); return; }
            if (!manual && Date.now() - ld('updT', 0) < 12 * 3600000) { showUpd(false); return; }
            if (manual) toast('\u062f\u0631 \u062d\u0627\u0644 \u0628\u0631\u0631\u0633\u06cc \u0622\u067e\u062f\u06cc\u062a...');
            http('https://api.github.com/repos/' + REPO + '/releases/latest', function (r) {
              if (r.status != 200) { if (manual) toast('\u062e\u0637\u0627 \u062f\u0631 \u0628\u0631\u0631\u0633\u06cc \u0622\u067e\u062f\u06cc\u062a (' + r.status + ')\u060c VPN \u0631\u0648 \u0686\u06a9 \u06a9\u0646'); showUpd(false); return; }
              try {
                var j = JSON.parse(r.body), n = parseInt(String(j.tag_name).replace(/\D/g, ''), 10) || 0;
                var a = (j.assets || []).filter(function (x) { return /\.apk$/.test(x.name); })[0];
                sv('upd', { n: n, url: a ? a.browser_download_url : j.html_url }); sv('updT', Date.now());
              } catch (e) {}
              showUpd(manual);
            });
          }
          function showUpd(manual) {
            var u = ld('upd', null), el = $('upd');
            if (u && u.n > BUILD) el.innerHTML = '<div class="box upd"><b>\ud83c\udd95 \u0646\u0633\u062e\u0647 \u062c\u062f\u06cc\u062f (v' + fa(u.n) + ') \u0622\u0645\u0627\u062f\u0647\u200c\u0633\u062a</b><br><button class="btn" onclick="getUpd()">\u2b07\ufe0f \u062f\u0627\u0646\u0644\u0648\u062f \u0648 \u0646\u0635\u0628</button></div>';
            else { el.innerHTML = ''; if (manual) toast('\u0622\u062e\u0631\u06cc\u0646 \u0646\u0633\u062e\u0647 \u0631\u0648 \u062f\u0627\u0631\u06cc \u2705'); }
          }
          function getUpd() { var u = ld('upd', null); if (!u) return; try { Android.openUrl(u.url); toast('\u0628\u0639\u062f \u0627\u0632 \u062f\u0627\u0646\u0644\u0648\u062f\u060c \u0641\u0627\u06cc\u0644 \u0631\u0648 \u0628\u0627\u0632 \u06a9\u0646 \u0648 \u00ab\u0628\u0647\u200c\u0631\u0648\u0632\u0631\u0633\u0627\u0646\u06cc\u00bb \u0631\u0648 \u0628\u0632\u0646'); } catch (e) { toast('\u0645\u0631\u0648\u0631\u06af\u0631 \u0628\u0627\u0632 \u0646\u0634\u062f'); } }

          /* start */
          render();
          remind();
          setTimeout(function () { checkUpdate(false); }, 1500);
          var LASTK = dk();
          setInterval(function () { if (dk() != LASTK) { LASTK = dk(); RV = null; render(); } }, 60000);
          </script>
          </body>
          </html>

          EOF_X
          sed -i "s/__BUILD__/$GITHUB_RUN_NUMBER/g" build.gradle src/main/assets/index.html
          sed -i "s#__REPO__#$GITHUB_REPOSITORY#g" src/main/assets/index.html

      - name: Build
        working-directory: app
        run: gradle assembleDebug --no-daemon

      - name: Rename APK
        run: cp app/build/outputs/apk/debug/*.apk lifeos.apk

      - uses: actions/upload-artifact@v4
        with:
          name: lifeos-apk
          path: lifeos.apk

      - uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ github.run_number }}
          name: Life OS v${{ github.run_number }}
          files: lifeos.apk

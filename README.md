name: Build APK
on: [push, workflow_dispatch]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"
      - name: Write project files
        run: |
          mkdir -p $(dirname settings.gradle.kts)
          cat > settings.gradle.kts <<'EOF'
          pluginManagement{repositories{google();mavenCentral();gradlePluginPortal()}}
          dependencyResolutionManagement{repositories{google();mavenCentral()}}
          rootProject.name="Padhai"
          include(":app")
          EOF
          mkdir -p $(dirname build.gradle.kts)
          cat > build.gradle.kts <<'EOF'
          plugins{
          id("com.android.application") version "8.5.2" apply false
          id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF
          mkdir -p $(dirname app/build.gradle.kts)
          cat > app/build.gradle.kts <<'EOF'
          plugins{id("com.android.application");id("org.jetbrains.kotlin.android")}
          android{
          namespace="com.pavan.padhai"
          compileSdk=34
          defaultConfig{applicationId="com.pavan.padhai";minSdk=26;targetSdk=33;versionCode=1;versionName="1.0"}
          compileOptions{sourceCompatibility=JavaVersion.VERSION_17;targetCompatibility=JavaVersion.VERSION_17}
          kotlinOptions{jvmTarget="17"}
          }
          EOF
          mkdir -p $(dirname app/src/main/AndroidManifest.xml)
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">
          <uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions"/>
          <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
          <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
          <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
          <application android:label="à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤¬à¥à¤²à¥‰à¤•à¤°" android:theme="@android:style/Theme.Material.Light.NoActionBar">
          <activity android:name=".MainActivity" android:exported="true"><intent-filter><action android:name="android.intent.action.MAIN"/><category android:name="android.intent.category.LAUNCHER"/></intent-filter></activity>
          <activity android:name=".BlockActivity" android:exported="false" android:excludeFromRecents="true"/>
          <service android:name=".FocusService" android:exported="false"/>
          </application>
          </manifest>
          EOF
          mkdir -p $(dirname app/src/main/java/com/pavan/padhai/MainActivity.kt)
          cat > app/src/main/java/com/pavan/padhai/MainActivity.kt <<'EOF'
          package com.pavan.padhai
          import android.app.Activity
          import android.content.Intent
          import android.net.Uri
          import android.os.Build
          import android.os.Bundle
          import android.provider.Settings
          import android.widget.*

          const val DEF = "com.google.android.youtube,com.instagram.android,com.facebook.katana,com.snapchat.android,com.twitter.android,com.instagram.lite,com.facebook.lite"

          class MainActivity : Activity() {
            override fun onCreate(b: Bundle?) {
              super.onCreate(b)
              if (Build.VERSION.SDK_INT >= 33) requestPermissions(arrayOf("android.permission.POST_NOTIFICATIONS"), 1)
              val p = getSharedPreferences("p", 0)
              val root = ScrollView(this)
              val l = LinearLayout(this); l.orientation = LinearLayout.VERTICAL; l.setPadding(40, 60, 40, 40)
              root.addView(l); setContentView(root)
              fun tv(t: String) = TextView(this).apply { text = t; textSize = 16f; setPadding(0, 24, 0, 6) }.also { l.addView(it) }
              fun btn(t: String, f: () -> Unit) = Button(this).apply { text = t; setOnClickListener { f() } }.also { l.addView(it) }
              tv("à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤¬à¥à¤²à¥‰à¤•à¤°").textSize = 26f
              btn("1. Usage Access à¤šà¤¾à¤²à¥‚ à¤•à¤°à¥‹") { startActivity(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS)) }
              btn("2. Overlay permission à¤¦à¥‹") { startActivity(Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:$packageName"))) }
              tv("à¤•à¤¿à¤¤à¤¨à¥‡ à¤®à¤¿à¤¨à¤Ÿ à¤ªà¤¢à¤¼à¤¨à¤¾ à¤¹à¥ˆ")
              val mins = EditText(this).apply { setText("25"); inputType = 2 }.also { l.addView(it) }
              tv("à¤¬à¥à¤²à¥‰à¤• à¤¹à¥‹à¤¨à¥‡ à¤µà¤¾à¤²à¥‡ à¤à¤ª (package à¤¨à¤¾à¤®, à¤•à¥‰à¤®à¤¾ à¤¸à¥‡ à¤…à¤²à¤—)")
              val bl = EditText(this).apply { setText(p.getString("block", DEF)) }.also { l.addView(it) }
              val strict = CheckBox(this).apply { text = "à¤¸à¥à¤Ÿà¥à¤°à¤¿à¤•à¥à¤Ÿ à¤®à¥‹à¤¡ (à¤¬à¥€à¤š à¤®à¥‡à¤‚ à¤°à¥‹à¤• à¤¨à¤¹à¥€à¤‚ à¤¸à¤•à¤¤à¥‡)"; isChecked = p.getBoolean("strict", false) }.also { l.addView(it) }
              val st = tv("")
              btn("à¤¶à¥à¤°à¥‚ à¤•à¤°à¥‹") {
                val m = mins.text.toString().toLongOrNull() ?: 25L
                p.edit().putLong("end", System.currentTimeMillis() + m * 60000).putString("block", bl.text.toString()).putBoolean("strict", strict.isChecked).apply()
                startForegroundService(Intent(this, FocusService::class.java))
                st.text = "à¤šà¤¾à¤²à¥‚! $m à¤®à¤¿à¤¨à¤Ÿ à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤•à¤°à¥‹à¥¤"
              }
              btn("à¤°à¥‹à¤•à¥‹") {
                if (p.getBoolean("strict", false) && System.currentTimeMillis() < p.getLong("end", 0)) st.text = "à¤¸à¥à¤Ÿà¥à¤°à¤¿à¤•à¥à¤Ÿ à¤®à¥‹à¤¡: à¤¸à¥‡à¤¶à¤¨ à¤ªà¥‚à¤°à¤¾ à¤¹à¥‹à¤¨à¥‡ à¤¤à¤• à¤¨à¤¹à¥€à¤‚ à¤°à¥à¤•à¥‡à¤—à¤¾!"
                else { p.edit().putLong("end", 0).apply(); stopService(Intent(this, FocusService::class.java)); st.text = "à¤¬à¤‚à¤¦ à¤•à¤¿à¤¯à¤¾à¥¤" }
              }
            }
          }
          EOF
          mkdir -p $(dirname app/src/main/java/com/pavan/padhai/FocusService.kt)
          cat > app/src/main/java/com/pavan/padhai/FocusService.kt <<'EOF'
          package com.pavan.padhai
          import android.app.*
          import android.app.usage.UsageEvents
          import android.app.usage.UsageStatsManager
          import android.content.Intent
          import android.os.*

          class FocusService : Service() {
            private val h = Handler(Looper.getMainLooper())
            private var cur = ""
            private var last = 0L
            private val loop = object : Runnable { override fun run() { check(); h.postDelayed(this, 1000) } }
            override fun onBind(i: Intent?): IBinder? = null
            override fun onStartCommand(i: Intent?, f: Int, s: Int): Int {
              val nm = getSystemService(NotificationManager::class.java)
              nm.createNotificationChannel(NotificationChannel("f", "Focus", NotificationManager.IMPORTANCE_LOW))
              startForeground(1, Notification.Builder(this, "f").setContentTitle("à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤šà¤¾à¤²à¥‚ à¤¹à¥ˆ").setSmallIcon(android.R.drawable.ic_lock_idle_lock).build())
              last = System.currentTimeMillis()
              h.removeCallbacks(loop); h.post(loop)
              return START_STICKY
            }
            override fun onDestroy() { h.removeCallbacks(loop) }
            private fun check() {
              val p = getSharedPreferences("p", 0)
              val now = System.currentTimeMillis()
              if (now >= p.getLong("end", 0)) { stopSelf(); return }
              val ev = getSystemService(UsageStatsManager::class.java).queryEvents(last, now)
              last = now
              val e = UsageEvents.Event()
              while (ev.hasNextEvent()) { ev.getNextEvent(e); if (e.eventType == UsageEvents.Event.ACTIVITY_RESUMED) cur = e.packageName }
              val bl = (p.getString("block", DEF) ?: DEF).split(",").map { it.trim() }
              if (cur in bl) startActivity(Intent(this, BlockActivity::class.java).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK))
            }
          }
          EOF
          mkdir -p $(dirname app/src/main/java/com/pavan/padhai/BlockActivity.kt)
          cat > app/src/main/java/com/pavan/padhai/BlockActivity.kt <<'EOF'
          package com.pavan.padhai
          import android.app.Activity
          import android.content.Intent
          import android.graphics.Color
          import android.os.Bundle
          import android.view.Gravity
          import android.widget.*

          class BlockActivity : Activity() {
            private val S = listOf("à¤«à¥‹à¤¨ à¤®à¥‡à¤‚ à¤•à¥à¤¯à¤¾ à¤•à¤° à¤°à¤¹à¤¾ à¤¥à¤¾? à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤›à¥‹à¤¡à¤¼à¤•à¤° à¤­à¤¾à¤— à¤—à¤¯à¤¾!", "à¤®à¤®à¥à¤®à¥€ à¤¦à¥‡à¤– à¤°à¤¹à¥€ à¤¹à¥ˆà¤‚â€¦ à¤šà¥à¤ªà¤šà¤¾à¤ª à¤µà¤¾à¤ªà¤¸ à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤•à¤°à¥¤", "à¤ªà¤¾à¤ªà¤¾ à¤•à¥‹ à¤¬à¤¤à¤¾à¤Šà¤? à¤šà¤², à¤•à¤¿à¤¤à¤¾à¤¬ à¤‰à¤ à¤¾à¥¤", "à¤à¤¸à¥‡ à¤°à¥€à¤² à¤¦à¥‡à¤–à¤¤à¤¾ à¤°à¤¹à¤¾ à¤¤à¥‹ à¤ªà¥ˆà¤¸à¥‡ à¤•à¤¬ à¤•à¤®à¤¾à¤à¤—à¤¾?")
            override fun onCreate(b: Bundle?) {
              super.onCreate(b)
              val t = TextView(this).apply { text = S.random(); textSize = 26f; setTextColor(Color.WHITE); gravity = Gravity.CENTER; setPadding(50, 50, 50, 50) }
              val bt = Button(this).apply { text = "à¤µà¤¾à¤ªà¤¸ à¤ªà¤¢à¤¼à¤¾à¤ˆ à¤ªà¤°"; setOnClickListener { home() } }
              val l = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; gravity = Gravity.CENTER; setBackgroundColor(Color.rgb(120, 15, 15)); addView(t); addView(bt) }
              setContentView(l)
            }
            private fun home() { startActivity(Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_HOME).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)); finish() }
            override fun onBackPressed() { home() }
          }
          EOF
      - run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: padhai-apk
          path: app/build/outputs/apk/debug/*.apk

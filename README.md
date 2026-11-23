name: Compilar Jarvis APK
on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: Crear proyecto Jarvis
        run: |
          mkdir -p app/src/main/java/com/jarvis/asistente

          cat > settings.gradle.kts <<'EOF'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "Jarvis"
          include(":app")
          EOF

          cat > build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF

          cat > gradle.properties <<'EOF'
          android.useAndroidX=true
          org.gradle.jvmargs=-Xmx2g
          EOF

          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.jarvis.asistente"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.jarvis.asistente"
                  minSdk = 24
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          EOF

          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.RECORD_AUDIO"/>
              <uses-permission android:name="android.permission.INTERNET"/>
              <queries>
                  <intent><action android:name="android.speech.RecognitionService"/></intent>
                  <intent><action android:name="android.intent.action.TTS_SERVICE"/></intent>
                  <intent>
                      <action android:name="android.intent.action.MAIN"/>
                      <category android:name="android.intent.category.LAUNCHER"/>
                  </intent>
              </queries>
              <application android:label="Jarvis"
                  android:theme="@android:style/Theme.DeviceDefault.Light.NoActionBar">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN"/>
                          <category android:name="android.intent.category.LAUNCHER"/>
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          EOF

          cat > app/src/main/java/com/jarvis/asistente/MainActivity.kt <<'EOF'
          package com.jarvis.asistente

          import android.Manifest
          import android.app.Activity
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.os.Bundle
          import android.speech.RecognitionListener
          import android.speech.RecognizerIntent
          import android.speech.SpeechRecognizer
          import android.speech.tts.TextToSpeech
          import android.text.InputType
          import android.widget.*
          import org.json.JSONArray
          import org.json.JSONObject
          import java.net.HttpURLConnection
          import java.net.URL
          import java.util.Locale

          class MainActivity : Activity() {
              private lateinit var tts: TextToSpeech
              private lateinit var sr: SpeechRecognizer
              private lateinit var keyBox: EditText
              private lateinit var log: TextView
              private lateinit var status: TextView
              private val history = JSONArray()

              override fun onCreate(b: Bundle?) {
                  super.onCreate(b)
                  val p = getSharedPreferences("j", MODE_PRIVATE)
                  val root = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(40, 80, 40, 40)
                  }
                  keyBox = EditText(this).apply {
                      hint = "API key de Anthropic"
                      inputType = InputType.TYPE_CLASS_TEXT or InputType.TYPE_TEXT_VARIATION_PASSWORD
                      setText(p.getString("k", ""))
                  }
                  status = TextView(this).apply { text = "Listo"; textSize = 16f; setPadding(0, 20, 0, 20) }
                  log = TextView(this).apply { textSize = 18f }
                  val scroll = ScrollView(this).apply { addView(log) }
                  val btn = Button(this).apply {
                      text = "Hablar con Jarvis"
                      setOnClickListener {
                          p.edit().putString("k", keyBox.text.toString().trim()).apply()
                          start()
                      }
                  }
                  root.addView(keyBox)
                  root.addView(status)
                  root.addView(btn)
                  root.addView(scroll, LinearLayout.LayoutParams(-1, 0, 1f))
                  setContentView(root)
                  tts = TextToSpeech(this) { tts.language = Locale("es", "ES") }
                  sr = SpeechRecognizer.createSpeechRecognizer(this)
              }

              private fun start() {
                  if (checkSelfPermission(Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), 1)
                      status.text = "Acepta el permiso y toca de nuevo"
                      return
                  }
                  tts.stop()
                  sr.setRecognitionListener(object : RecognitionListener {
                      override fun onReadyForSpeech(p: Bundle?) { status.text = "Escuchando..." }
                      override fun onBeginningOfSpeech() {}
                      override fun onRmsChanged(v: Float) {}
                      override fun onBufferReceived(b: ByteArray?) {}
                      override fun onEndOfSpeech() {}
                      override fun onError(e: Int) { status.text = "No te escuche (error $e). Toca de nuevo." }
                      override fun onResults(r: Bundle?) {
                          val t = r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
                          if (t != null) handle(t)
                      }
                      override fun onPartialResults(p: Bundle?) {}
                      override fun onEvent(t: Int, p: Bundle?) {}
                  })
                  sr.startListening(
                      Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
                          .putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                          .putExtra(RecognizerIntent.EXTRA_LANGUAGE, "es-ES")
                  )
              }

              private fun say(t: String) {
                  log.append("Jarvis: $t\n\n")
                  tts.speak(t, TextToSpeech.QUEUE_FLUSH, null, "j")
              }

              private fun handle(t: String) {
                  log.append("Tu: $t\n\n")
                  val l = t.lowercase()
                  if (l.startsWith("abre ")) { openApp(l.removePrefix("abre ").trim()); return }
                  status.text = "Pensando..."
                  ask(t)
              }

              private fun openApp(n: String) {
                  val pm = packageManager
                  val apps = pm.queryIntentActivities(
                      Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER), 0)
                  val a = apps.firstOrNull { it.loadLabel(pm).toString().lowercase().contains(n) }
                  val i = a?.let { pm.getLaunchIntentForPackage(it.activityInfo.packageName) }
                  if (i != null) { say("Abriendo $n"); startActivity(i) }
                  else say("No encontre la app $n")
              }

              private fun ask(t: String) {
                  history.put(JSONObject().put("role", "user").put("content", t))
                  val key = keyBox.text.toString().trim()
                  Thread {
                      try {
                          val body = JSONObject()
                              .put("model", "claude-haiku-5-5")
                              .put("max_tokens", 400)
                              .put("system", "Eres Jarvis, asistente de voz. Responde en espanol, en maximo 3 frases cortas, sin markdown ni listas.")
                              .put("messages", history)
                          val c = URL("https://api.anthropic.com/v1/messages").openConnection() as HttpURLConnection
                          c.requestMethod = "POST"
                          c.doOutput = true
                          c.setRequestProperty("x-api-key", key)
                          c.setRequestProperty("anthropic-version", "2023-06-01")
                          c.setRequestProperty("content-type", "application/json")
                          c.outputStream.use { it.write(body.toString().toByteArray()) }
                          val code = c.responseCode
                          val txt = (if (code < 400) c.inputStream else c.errorStream).bufferedReader().readText()
                          if (code >= 400) throw Exception(txt)
                          val ans = JSONObject(txt).getJSONArray("content").getJSONObject(0).getString("text")
                          history.put(JSONObject().put("role", "assistant").put("content", ans))
                          while (history.length() > 20) { history.remove(0); history.remove(0) }
                          runOnUiThread { status.text = "Listo"; say(ans) }
                      } catch (e: Exception) {
                          history.remove(history.length() - 1)
                          runOnUiThread { status.text = "Error: ${e.message}".take(300) }
                      }
                  }.start()
              }

              override fun onDestroy() {
                  sr.destroy()
                  tts.shutdown()
                  super.onDestroy()
              }
          }
          EOF
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.9
      - name: Compilar
        run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: Jarvis-APK
          path: app/build/outputs/apk/debug/app-debug.apk

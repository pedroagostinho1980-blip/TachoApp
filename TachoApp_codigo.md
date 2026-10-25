# TachoApp: código completo

Cada secção é um ficheiro. Cria a mesma estrutura de pastas no Android Studio e cola o conteúdo. (O gradle-wrapper não está aqui: o Android Studio cria-o.)

## `README.md`

```markdown
# Tempos de Condução (Android)

App para o condutor de pesados controlar os tempos de condução e descanso segundo o
Regulamento (CE) n.º 561/2006. Kotlin + Jetpack Compose + Room.

> Ajuda ao condutor. **Não substitui o tacógrafo** nem tem valor legal.

## Como abrir

1. Abrir esta pasta no **Android Studio** (Ladybug ou mais recente, JDK 17).
2. Deixar o Gradle sincronizar (se pedir, aceitar as atualizações de versões sugeridas).
3. Correr no telemóvel ou emulador (Android 8.0+, API 26).

Testes do motor de regras (não precisam de Android): `./gradlew :core:test`

## Estrutura

| Módulo | Conteúdo |
|---|---|
| `core/` | Motor de regras em Kotlin puro (`RulesEngine`), resumos diários/semanais e testes |
| `app/data` | Room: registos de atividades (`ActivityEntity`, `ActivityRepository`) |
| `app/service` | Serviço em primeiro plano + notificações e alertas |
| `app/ui` | Ecrãs Compose: **Hoje** (botões, contadores, prazos) e **Histórico** (editar, adicionar, exportar) |
| `app/export` | Exportação CSV e PDF |

## Regras implementadas

- Condução contínua máx. 4h30; pausa de 45 min, ou 15 min + 30 min (por esta ordem)
- Condução diária 9h, com 2 prolongamentos a 10h por semana
- Condução semanal 56h e quinzenal 90h (semana fixa, segunda a domingo)
- Descanso diário 11h, reduzido 9h (máx. 3 entre descansos semanais) ou fracionado 3h + 9h
- Descanso semanal (≥ 24h conta como semanal) e prazo de 6 períodos de 24h
- Prazos para iniciar o descanso diário e semanal
- Alertas escalonados (informação → aviso → limite) com o serviço ativo fora do descanso

## Pressupostos e limites conhecidos

- Intervalos sem registo e o tempo depois da última atividade fechada contam como descanso.
- Só a atividade «Descanso» conta como pausa; outro trabalho e disponibilidade não.
- A semana fixa usa UTC (como o tacógrafo); os horários apresentados usam o fuso do telemóvel.
- Sem descanso registado, os prazos contam desde o primeiro registo (a app avisa).
- Ainda **não** modelado: compensação de descansos semanais reduzidos, ferry/comboio,
  dois condutores, exceções do regulamento, transporte internacional de passageiros.
- O serviço em primeiro plano usa o tipo `specialUse`; a publicação na Google Play exige
  justificação na declaração de serviços em primeiro plano.

## Próximos passos sugeridos

1. Compensação de descansos semanais reduzidos.
2. Ecrã de definições (limites de aviso, idioma, fuso).
3. Cópia de segurança e importação do histórico.
4. Testes instrumentados do serviço e da interface.
```

## `.gitignore`

```
.gradle/
build/
local.properties
.idea/
*.iml
captures/
.externalNativeBuild/
.cxx/
```

## `app/build.gradle.kts`

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("com.google.devtools.ksp")
}

android {
    namespace = "pt.tacho.app"
    compileSdk = 35

    defaultConfig {
        applicationId = "pt.tacho.app"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }
}

ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}

dependencies {
    implementation(project(":core"))

    implementation("androidx.core:core-ktx:1.15.0")
    implementation("androidx.activity:activity-compose:1.9.3")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.7")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")

    val composeBom = platform("androidx.compose:compose-bom:2024.11.00")
    implementation(composeBom)
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling")

    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0")

    testImplementation(kotlin("test"))
}
```

## `app/proguard-rules.pro`

```
# Room e Compose trazem as suas próprias regras de consumidor.
```

## `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.VIBRATE" />

    <application
        android:name=".TachoApplication"
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.Tacho">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".service.TrackingService"
            android:exported="false"
            android:foregroundServiceType="specialUse">
            <property
                android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                android:value="driver_time_tracking" />
        </service>

        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
    </application>
</manifest>
```

## `app/src/main/java/pt/tacho/app/MainActivity.kt`

```kotlin
package pt.tacho.app

import android.Manifest
import android.os.Build
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.activity.result.contract.ActivityResultContracts
import androidx.activity.viewModels
import androidx.compose.foundation.layout.padding
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.History
import androidx.compose.material.icons.filled.Timer
import androidx.compose.material3.Icon
import androidx.compose.material3.NavigationBar
import androidx.compose.material3.NavigationBarItem
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.res.stringResource
import pt.tacho.app.ui.HistoryScreen
import pt.tacho.app.ui.TachoTheme
import pt.tacho.app.ui.TachoViewModel
import pt.tacho.app.ui.TodayScreen

class MainActivity : ComponentActivity() {
    private val vm: TachoViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TachoTheme { TachoApp(vm) }
        }
    }

    override fun onStart() {
        super.onStart()
        vm.syncService()
    }
}

@Composable
private fun TachoApp(vm: TachoViewModel) {
    var tab by rememberSaveable { mutableIntStateOf(0) }
    RequestNotificationPermission()

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    selected = tab == 0,
                    onClick = { tab = 0 },
                    icon = { Icon(Icons.Filled.Timer, contentDescription = null) },
                    label = { Text(stringResource(R.string.tab_today)) },
                )
                NavigationBarItem(
                    selected = tab == 1,
                    onClick = { tab = 1 },
                    icon = { Icon(Icons.Filled.History, contentDescription = null) },
                    label = { Text(stringResource(R.string.tab_history)) },
                )
            }
        },
    ) { padding ->
        val modifier = Modifier.padding(padding)
        if (tab == 0) TodayScreen(vm, modifier) else HistoryScreen(vm, modifier)
    }
}

/** Pede a permissão de notificações (Android 13+) uma vez, ao abrir a app. */
@Composable
private fun RequestNotificationPermission() {
    if (Build.VERSION.SDK_INT < 33) return
    val launcher = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) { }
    LaunchedEffect(Unit) { launcher.launch(Manifest.permission.POST_NOTIFICATIONS) }
}
```

## `app/src/main/java/pt/tacho/app/TachoApplication.kt`

```kotlin
package pt.tacho.app

import android.app.Application
import android.content.Context
import pt.tacho.app.data.ActivityRepository
import pt.tacho.app.data.TachoDatabase
import pt.tacho.app.service.Notifications

class TachoApplication : Application() {
    val repository: ActivityRepository by lazy { ActivityRepository(TachoDatabase.build(this)) }

    override fun onCreate() {
        super.onCreate()
        Notifications.createChannels(this)
    }
}

val Context.repository: ActivityRepository
    get() = (applicationContext as TachoApplication).repository
```

## `app/src/main/java/pt/tacho/app/data/ActivityDao.kt`

```kotlin
package pt.tacho.app.data

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface ActivityDao {
    @Insert
    suspend fun insert(activity: ActivityEntity): Long

    @Update
    suspend fun update(activity: ActivityEntity)

    @Delete
    suspend fun delete(activity: ActivityEntity)

    @Query("SELECT * FROM activities ORDER BY startMs DESC")
    fun observeAll(): Flow<List<ActivityEntity>>

    @Query("SELECT * FROM activities WHERE endMs IS NULL OR endMs >= :since ORDER BY startMs ASC")
    fun observeSince(since: Long): Flow<List<ActivityEntity>>

    @Query("SELECT * FROM activities WHERE endMs IS NULL OR endMs >= :since ORDER BY startMs ASC")
    suspend fun getSince(since: Long): List<ActivityEntity>

    @Query("SELECT * FROM activities ORDER BY startMs ASC")
    suspend fun getAll(): List<ActivityEntity>

    @Query("SELECT * FROM activities WHERE endMs IS NULL ORDER BY startMs DESC")
    suspend fun getOpen(): List<ActivityEntity>

    @Query("UPDATE activities SET endMs = :endMs WHERE endMs IS NULL")
    suspend fun closeOpen(endMs: Long)
}
```

## `app/src/main/java/pt/tacho/app/data/ActivityEntity.kt`

```kotlin
package pt.tacho.app.data

import androidx.room.Entity
import androidx.room.Index
import androidx.room.PrimaryKey
import pt.tacho.core.Activity
import pt.tacho.core.ActivityType

@Entity(tableName = "activities", indices = [Index("startMs")])
data class ActivityEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    /** Nome de [ActivityType]. */
    val type: String,
    val startMs: Long,
    /** Nulo = em curso. */
    val endMs: Long?,
) {
    val activityType: ActivityType get() = ActivityType.valueOf(type)

    fun toCore() = Activity(activityType, startMs, endMs)
}
```

## `app/src/main/java/pt/tacho/app/data/ActivityRepository.kt`

```kotlin
package pt.tacho.app.data

import androidx.room.withTransaction
import kotlinx.coroutines.flow.Flow
import pt.tacho.core.ActivityType

class ActivityRepository(private val db: TachoDatabase) {
    private val dao = db.activityDao()

    /** Janela usada pelo motor: cobre a semana atual, a anterior e os prazos de descanso. */
    private fun recentSince(): Long = System.currentTimeMillis() - RECENT_WINDOW_MS

    fun observeRecent(): Flow<List<ActivityEntity>> = dao.observeSince(recentSince())

    fun observeAll(): Flow<List<ActivityEntity>> = dao.observeAll()

    suspend fun recent(): List<ActivityEntity> = dao.getSince(recentSince())

    suspend fun all(): List<ActivityEntity> = dao.getAll()

    suspend fun currentType(): ActivityType? =
        dao.getOpen().firstOrNull()?.activityType

    /** Fecha a atividade em curso e abre a nova. Não faz nada se já for esta a atividade em curso. */
    suspend fun switchTo(type: ActivityType, now: Long = System.currentTimeMillis()) {
        db.withTransaction {
            val open = dao.getOpen()
            if (open.size == 1 && open[0].activityType == type) return@withTransaction
            dao.closeOpen(now)
            dao.insert(ActivityEntity(type = type.name, startMs = now, endMs = null))
        }
    }

    /** Insere ou atualiza um registo manual. Devolve false se o intervalo for inválido. */
    suspend fun save(entry: ActivityEntity, now: Long = System.currentTimeMillis()): Boolean {
        val end = entry.endMs
        if (end != null && (end <= entry.startMs || end > now)) return false
        if (entry.startMs >= now) return false
        if (entry.id == 0L) dao.insert(entry) else dao.update(entry)
        return true
    }

    suspend fun delete(entry: ActivityEntity) = dao.delete(entry)

    companion object {
        const val RECENT_WINDOW_MS = 35L * 24 * 60 * 60 * 1000
    }
}
```

## `app/src/main/java/pt/tacho/app/data/TachoDatabase.kt`

```kotlin
package pt.tacho.app.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [ActivityEntity::class], version = 1, exportSchema = true)
abstract class TachoDatabase : RoomDatabase() {
    abstract fun activityDao(): ActivityDao

    companion object {
        fun build(context: Context): TachoDatabase =
            Room.databaseBuilder(context, TachoDatabase::class.java, "tacho.db").build()
    }
}
```

## `app/src/main/java/pt/tacho/app/export/Exporter.kt`

```kotlin
package pt.tacho.app.export

import android.content.Context
import android.content.Intent
import android.graphics.Paint
import android.graphics.Typeface
import android.graphics.pdf.PdfDocument
import androidx.core.content.FileProvider
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import pt.tacho.app.R
import pt.tacho.app.data.ActivityRepository
import pt.tacho.app.util.activityLabel
import pt.tacho.app.util.formatDate
import pt.tacho.app.util.formatDateTime
import pt.tacho.app.util.formatHm
import pt.tacho.core.summarizeByDay
import pt.tacho.core.summarizeDrivingByWeek
import java.io.File
import java.time.Instant
import java.time.ZoneId
import java.time.ZoneOffset
import java.time.format.DateTimeFormatter

class Exporter(private val context: Context, private val repo: ActivityRepository) {

    private fun exportDir(): File = File(context.cacheDir, "exports").apply { mkdirs() }

    private fun stamp(now: Long): String =
        DateTimeFormatter.ofPattern("yyyyMMdd_HHmm").format(Instant.ofEpochMilli(now).atZone(ZoneId.systemDefault()))

    /** CSV com ";" como separador e BOM UTF-8, para abrir bem no Excel em português. */
    suspend fun csv(): File = withContext(Dispatchers.IO) {
        val now = System.currentTimeMillis()
        val file = File(exportDir(), "tempos_${stamp(now)}.csv")
        val iso = DateTimeFormatter.ISO_OFFSET_DATE_TIME
        file.bufferedWriter(Charsets.UTF_8).use { w ->
            w.write("﻿")
            w.appendLine("tipo;inicio;fim;minutos")
            for (a in repo.all()) {
                val end = a.endMs
                val minutes = ((end ?: now) - a.startMs) / 60_000
                val start = iso.format(Instant.ofEpochMilli(a.startMs).atZone(ZoneId.systemDefault()))
                val endText = end?.let { iso.format(Instant.ofEpochMilli(it).atZone(ZoneId.systemDefault())) } ?: ""
                w.appendLine("${context.getString(activityLabel(a.activityType))};$start;$endText;$minutes")
            }
        }
        file
    }

    /** PDF A4 com resumo diário (últimos 28 dias) e condução por semana. */
    suspend fun pdf(): File = withContext(Dispatchers.IO) {
        val now = System.currentTimeMillis()
        val activities = repo.all().map { it.toCore() }
        val days = summarizeByDay(activities, ZoneId.systemDefault(), now).takeLast(28)
        val weeks = summarizeDrivingByWeek(activities, ZoneOffset.UTC, now).takeLast(8)

        val doc = PdfDocument()
        val w = PdfWriter(doc)
        val cols = listOf(40f, 150f, 250f, 350f, 450f)

        w.title(context.getString(R.string.report_title))
        w.text(context.getString(R.string.report_generated, formatDateTime(now)))
        w.gap(10f)
        w.row(
            listOf(
                context.getString(R.string.report_col_date),
                context.getString(R.string.report_col_driving),
                context.getString(R.string.report_col_work),
                context.getString(R.string.report_col_availability),
                context.getString(R.string.report_col_rest),
            ),
            cols, strong = true,
        )
        for (d in days) {
            w.row(
                listOf(
                    formatDate(d.date),
                    formatHm(d.drivingMs),
                    formatHm(d.otherWorkMs),
                    formatHm(d.availabilityMs),
                    formatHm(d.restMs),
                ),
                cols,
            )
        }

        w.gap(16f)
        w.text(context.getString(R.string.report_weeks_title), strong = true)
        for ((monday, driving) in weeks) {
            w.row(
                listOf(context.getString(R.string.report_week_of, formatDate(monday)), formatHm(driving)),
                listOf(40f, 250f),
            )
        }

        w.gap(16f)
        w.text(context.getString(R.string.report_disclaimer))
        w.finish()

        val file = File(exportDir(), "relatorio_${stamp(now)}.pdf")
        file.outputStream().use { doc.writeTo(it) }
        doc.close()
        file
    }

    fun share(file: File, mime: String): Intent {
        val uri = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
        return Intent.createChooser(
            Intent(Intent.ACTION_SEND)
                .setType(mime)
                .putExtra(Intent.EXTRA_STREAM, uri)
                .addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION),
            context.getString(R.string.export_share),
        )
    }

    private class PdfWriter(private val doc: PdfDocument) {
        private val margin = 40f
        private val pageHeight = 842
        private val normal = Paint().apply { textSize = 10f; isAntiAlias = true }
        private val bold = Paint(normal).apply { typeface = Typeface.DEFAULT_BOLD }
        private val heading = Paint(bold).apply { textSize = 16f }
        private var page: PdfDocument.Page? = null
        private var pageNo = 0
        private var y = 0f

        private fun ensure(height: Float) {
            if (page == null || y + height > pageHeight - margin) {
                finish()
                pageNo++
                page = doc.startPage(PdfDocument.PageInfo.Builder(595, pageHeight, pageNo).create())
                y = margin
            }
        }

        fun title(text: String) {
            ensure(30f)
            page!!.canvas.drawText(text, margin, y + 16f, heading)
            y += 28f
        }

        fun text(text: String, strong: Boolean = false) {
            ensure(16f)
            page!!.canvas.drawText(text, margin, y + 10f, if (strong) bold else normal)
            y += 16f
        }

        fun row(cells: List<String>, xs: List<Float>, strong: Boolean = false) {
            ensure(16f)
            cells.forEachIndexed { i, c -> page!!.canvas.drawText(c, xs[i], y + 10f, if (strong) bold else normal) }
            y += 16f
        }

        fun gap(height: Float) {
            y += height
        }

        fun finish() {
            page?.let { doc.finishPage(it) }
            page = null
        }
    }
}
```

## `app/src/main/java/pt/tacho/app/service/Notifications.kt`

```kotlin
package pt.tacho.app.service

import android.annotation.SuppressLint
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import pt.tacho.app.MainActivity
import pt.tacho.app.R
import pt.tacho.app.util.activityLabel
import pt.tacho.app.util.alertMessage
import pt.tacho.app.util.alertTitle
import pt.tacho.app.util.formatHm
import pt.tacho.app.util.limitLabel
import pt.tacho.core.Alert
import pt.tacho.core.AlertCode
import pt.tacho.core.AlertLevel
import pt.tacho.core.ActivityType
import pt.tacho.core.DriverStatus

object Notifications {
    const val CHANNEL_ONGOING = "ongoing"
    const val CHANNEL_ALERTS = "alerts"
    const val ONGOING_ID = 1
    private const val ALERT_BASE_ID = 100

    fun createChannels(context: Context) {
        val nm = context.getSystemService(NotificationManager::class.java)
        nm.createNotificationChannel(
            NotificationChannel(
                CHANNEL_ONGOING,
                context.getString(R.string.channel_ongoing),
                NotificationManager.IMPORTANCE_LOW,
            ).apply { description = context.getString(R.string.channel_ongoing_desc) },
        )
        nm.createNotificationChannel(
            NotificationChannel(
                CHANNEL_ALERTS,
                context.getString(R.string.channel_alerts),
                NotificationManager.IMPORTANCE_HIGH,
            ).apply {
                description = context.getString(R.string.channel_alerts_desc)
                enableVibration(true)
                vibrationPattern = longArrayOf(0, 400, 200, 400)
            },
        )
    }

    private fun contentIntent(context: Context): PendingIntent = PendingIntent.getActivity(
        context,
        0,
        Intent(context, MainActivity::class.java)
            .addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_CLEAR_TOP),
        PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
    )

    /** Notificação inicial, exigida nos primeiros segundos de um serviço em primeiro plano. */
    fun placeholder(context: Context): Notification =
        NotificationCompat.Builder(context, CHANNEL_ONGOING)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(context.getString(R.string.app_name))
            .setContentText(context.getString(R.string.notif_ongoing_placeholder))
            .setOngoing(true)
            .setOnlyAlertOnce(true)
            .setContentIntent(contentIntent(context))
            .build()

    fun ongoing(context: Context, status: DriverStatus): Notification {
        val type = status.currentActivity
        val title = type?.let { context.getString(activityLabel(it)) } ?: context.getString(R.string.app_name)
        val text = if (type == ActivityType.DRIVING) {
            context.getString(R.string.notif_ongoing_text, formatHm(status.canDriveMs)) +
                " · " + context.getString(limitLabel(status.limitingFactor))
        } else {
            context.getString(R.string.notif_ongoing_placeholder)
        }
        return NotificationCompat.Builder(context, CHANNEL_ONGOING)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(text)
            .setOngoing(true)
            .setOnlyAlertOnce(true)
            .setShowWhen(true)
            .setUsesChronometer(true)
            .setWhen(status.currentActivityStartMs ?: status.nowMs)
            .setContentIntent(contentIntent(context))
            .build()
    }

    fun alert(context: Context, alert: Alert): Notification =
        NotificationCompat.Builder(context, CHANNEL_ALERTS)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(alertTitle(context, alert.level))
            .setContentText(alertMessage(context, alert))
            .setCategory(NotificationCompat.CATEGORY_ALARM)
            .setPriority(
                if (alert.level == AlertLevel.INFO) NotificationCompat.PRIORITY_DEFAULT
                else NotificationCompat.PRIORITY_HIGH,
            )
            .setAutoCancel(true)
            .setContentIntent(contentIntent(context))
            .build()

    fun alertId(code: AlertCode): Int = ALERT_BASE_ID + code.ordinal

    @SuppressLint("MissingPermission") // sem permissão POST_NOTIFICATIONS o sistema ignora silenciosamente
    fun post(context: Context, id: Int, notification: Notification) {
        NotificationManagerCompat.from(context).notify(id, notification)
    }

    fun cancelAlerts(context: Context) {
        val nm = NotificationManagerCompat.from(context)
        AlertCode.values().forEach { nm.cancel(alertId(it)) }
    }
}
```

## `app/src/main/java/pt/tacho/app/service/TrackingService.kt`

```kotlin
package pt.tacho.app.service

import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.pm.ServiceInfo
import android.os.Build
import android.os.IBinder
import androidx.core.content.ContextCompat
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import pt.tacho.app.repository
import pt.tacho.core.ActivityType
import pt.tacho.core.AlertCode
import pt.tacho.core.AlertLevel
import pt.tacho.core.DriverStatus
import pt.tacho.core.RulesEngine

/**
 * Serviço em primeiro plano ativo enquanto o condutor NÃO está em descanso. Mantém a notificação
 * com o tempo restante e dispara alertas quando um limite se aproxima ou é atingido.
 */
class TrackingService : Service() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    private val engine = RulesEngine()
    private val notifiedLevels = mutableMapOf<AlertCode, AlertLevel>()
    private var job: Job? = null

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        goForeground()
        if (job?.isActive != true) job = scope.launch { loop() }
        return START_STICKY
    }

    private fun goForeground() {
        val notification = Notifications.placeholder(this)
        if (Build.VERSION.SDK_INT >= 34) {
            startForeground(
                Notifications.ONGOING_ID,
                notification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE,
            )
        } else {
            startForeground(Notifications.ONGOING_ID, notification)
        }
    }

    private suspend fun loop() {
        while (scope.isActive) {
            val now = System.currentTimeMillis()
            val status = engine.evaluate(repository.recent().map { it.toCore() }, now)

            val current = status.currentActivity
            if (current == null || current == ActivityType.REST) {
                stopSelf()
                return
            }

            Notifications.post(this, Notifications.ONGOING_ID, Notifications.ongoing(this, status))
            fireAlerts(status)
            delay(TICK_MS)
        }
    }

    /** Notifica quando um alerta aparece ou sobe de nível; limpa o histórico quando desaparece. */
    private fun fireAlerts(status: DriverStatus) {
        val active = status.alerts.associateBy { it.code }
        notifiedLevels.keys.retainAll(active.keys)
        for (alert in status.alerts) {
            val previous = notifiedLevels[alert.code]
            if (previous == null || alert.level > previous) {
                Notifications.post(this, Notifications.alertId(alert.code), Notifications.alert(this, alert))
                notifiedLevels[alert.code] = alert.level
            }
        }
    }

    override fun onDestroy() {
        scope.cancel()
        Notifications.cancelAlerts(this)
        super.onDestroy()
    }

    companion object {
        private const val TICK_MS = 15_000L

        /** Liga o serviço se a atividade atual não for descanso; desliga-o caso contrário. */
        fun sync(context: Context, current: ActivityType?) {
            val intent = Intent(context, TrackingService::class.java)
            if (current != null && current != ActivityType.REST) {
                ContextCompat.startForegroundService(context, intent)
            } else {
                context.stopService(intent)
            }
        }
    }
}
```

## `app/src/main/java/pt/tacho/app/ui/HistoryScreen.kt`

```kotlin
package pt.tacho.app.ui

import android.app.DatePickerDialog
import android.app.TimePickerDialog
import android.content.Context
import android.widget.Toast
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.ExtendedFloatingActionButton
import androidx.compose.material3.FilterChip
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import kotlinx.coroutines.launch
import pt.tacho.app.R
import pt.tacho.app.data.ActivityEntity
import pt.tacho.app.util.activityLabel
import pt.tacho.app.util.formatClock
import pt.tacho.app.util.formatDateTime
import pt.tacho.app.util.formatDayHeader
import pt.tacho.app.util.formatHm
import pt.tacho.core.ActivityType
import pt.tacho.core.summarizeByDay
import java.time.Instant
import java.time.ZoneId
import java.util.Calendar

@Composable
fun HistoryScreen(vm: TachoViewModel, modifier: Modifier = Modifier) {
    val context = LocalContext.current
    val scope = rememberCoroutineScope()
    val items by vm.history.collectAsStateWithLifecycle()
    var editing by remember { mutableStateOf<ActivityEntity?>(null) }

    val zone = ZoneId.systemDefault()
    val grouped = items.groupBy { Instant.ofEpochMilli(it.startMs).atZone(zone).toLocalDate() }
    val totals = remember(items) {
        summarizeByDay(items.map { it.toCore() }, zone, System.currentTimeMillis()).associateBy { it.date }
    }

    Box(modifier.fillMaxSize()) {
        Column(Modifier.fillMaxSize()) {
            Row(
                Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp),
            ) {
                OutlinedButton(onClick = {
                    scope.launch { shareOrToast(context, vm.exportCsv()) }
                }) { Text(stringResource(R.string.export_csv)) }
                OutlinedButton(onClick = {
                    scope.launch { shareOrToast(context, vm.exportPdf()) }
                }) { Text(stringResource(R.string.export_pdf)) }
            }

            if (items.isEmpty()) {
                Text(
                    stringResource(R.string.history_empty),
                    modifier = Modifier.padding(16.dp),
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                )
            }

            LazyColumn(
                contentPadding = PaddingValues(start = 16.dp, end = 16.dp, bottom = 96.dp),
                verticalArrangement = Arrangement.spacedBy(2.dp),
            ) {
                grouped.forEach { (date, entries) ->
                    item(key = "h$date") {
                        Column(Modifier.padding(top = 16.dp, bottom = 4.dp)) {
                            Text(
                                formatDayHeader(date),
                                style = MaterialTheme.typography.titleMedium,
                                fontWeight = FontWeight.SemiBold,
                            )
                            totals[date]?.let {
                                Text(
                                    stringResource(
                                        R.string.day_totals,
                                        formatHm(it.drivingMs),
                                        formatHm(it.otherWorkMs),
                                    ),
                                    style = MaterialTheme.typography.bodySmall,
                                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                                )
                            }
                        }
                    }
                    items(entries, key = { it.id }) { entry ->
                        EntryRow(entry, onClick = { editing = entry })
                    }
                }
            }
        }

        ExtendedFloatingActionButton(
            onClick = {
                val now = System.currentTimeMillis()
                editing = ActivityEntity(type = ActivityType.DRIVING.name, startMs = now - 3_600_000L, endMs = now)
            },
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(16.dp),
            icon = { Icon(Icons.Filled.Add, contentDescription = null) },
            text = { Text(stringResource(R.string.add_entry)) },
        )
    }

    editing?.let { entry ->
        EditEntryDialog(
            initial = entry,
            onDismiss = { editing = null },
            onSave = { updated ->
                vm.save(updated) { ok ->
                    if (ok) editing = null
                    else Toast.makeText(context, R.string.error_range, Toast.LENGTH_LONG).show()
                }
            },
            onDelete = {
                vm.delete(entry)
                editing = null
            },
        )
    }
}

private fun shareOrToast(context: Context, intent: android.content.Intent?) {
    if (intent != null) context.startActivity(intent)
    else Toast.makeText(context, R.string.export_failed, Toast.LENGTH_LONG).show()
}

@Composable
private fun EntryRow(entry: ActivityEntity, onClick: () -> Unit) {
    val end = entry.endMs
    val duration = (end ?: System.currentTimeMillis()) - entry.startMs
    Row(
        Modifier
            .fillMaxWidth()
            .clickable(onClick = onClick)
            .padding(vertical = 10.dp),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Spacer(
            Modifier
                .size(12.dp)
                .clip(CircleShape)
                .background(activityColor(entry.activityType)),
        )
        Column(Modifier.padding(start = 12.dp)) {
            Text(stringResource(activityLabel(entry.activityType)), style = MaterialTheme.typography.bodyLarge)
            Text(
                text = "${formatClock(entry.startMs)} – ${end?.let { formatClock(it) } ?: stringResource(R.string.entry_ongoing)}",
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
            )
        }
        Box(Modifier.weight(1f), contentAlignment = Alignment.CenterEnd) {
            Text(formatHm(duration), style = MaterialTheme.typography.titleSmall, fontWeight = FontWeight.SemiBold)
        }
    }
}

@Composable
private fun EditEntryDialog(
    initial: ActivityEntity,
    onDismiss: () -> Unit,
    onSave: (ActivityEntity) -> Unit,
    onDelete: () -> Unit,
) {
    val context = LocalContext.current
    var type by remember { mutableStateOf(initial.activityType) }
    var start by remember { mutableStateOf(initial.startMs) }
    var end by remember { mutableStateOf(initial.endMs) }
    val ongoing = initial.id != 0L && initial.endMs == null

    AlertDialog(
        onDismissRequest = onDismiss,
        title = {
            Text(stringResource(if (initial.id == 0L) R.string.new_entry else R.string.edit_entry))
        },
        text = {
            Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
                Text(stringResource(R.string.entry_type), style = MaterialTheme.typography.labelLarge)
                ActivityType.values().toList().chunked(2).forEach { pair ->
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        pair.forEach { t ->
                            FilterChip(
                                selected = t == type,
                                onClick = { type = t },
                                label = { Text(stringResource(activityLabel(t))) },
                            )
                        }
                    }
                }
                Text(stringResource(R.string.entry_start), style = MaterialTheme.typography.labelLarge)
                OutlinedButton(onClick = { pickDateTime(context, start) { start = it } }) {
                    Text(formatDateTime(start))
                }
                Text(stringResource(R.string.entry_end), style = MaterialTheme.typography.labelLarge)
                if (ongoing) {
                    Text(stringResource(R.string.entry_ongoing))
                } else {
                    OutlinedButton(onClick = { pickDateTime(context, end ?: start) { end = it } }) {
                        Text(end?.let { formatDateTime(it) } ?: "—")
                    }
                }
            }
        },
        confirmButton = {
            TextButton(onClick = {
                onSave(initial.copy(type = type.name, startMs = start, endMs = if (ongoing) null else end))
            }) { Text(stringResource(R.string.save)) }
        },
        dismissButton = {
            Row {
                if (initial.id != 0L) {
                    TextButton(onClick = onDelete) {
                        Text(stringResource(R.string.delete), color = MaterialTheme.colorScheme.error)
                    }
                }
                TextButton(onClick = onDismiss) { Text(stringResource(R.string.cancel)) }
            }
        },
    )
}

/** Seletor de data seguido de hora, com os diálogos nativos do Android. */
private fun pickDateTime(context: Context, initialMs: Long, onPicked: (Long) -> Unit) {
    val cal = Calendar.getInstance().apply { timeInMillis = initialMs }
    DatePickerDialog(
        context,
        { _, year, month, day ->
            TimePickerDialog(
                context,
                { _, hour, minute ->
                    val picked = Calendar.getInstance().apply {
                        set(year, month, day, hour, minute, 0)
                        set(Calendar.MILLISECOND, 0)
                    }
                    onPicked(picked.timeInMillis)
                },
                cal.get(Calendar.HOUR_OF_DAY),
                cal.get(Calendar.MINUTE),
                true,
            ).show()
        },
        cal.get(Calendar.YEAR),
        cal.get(Calendar.MONTH),
        cal.get(Calendar.DAY_OF_MONTH),
    ).show()
}
```

## `app/src/main/java/pt/tacho/app/ui/TachoViewModel.kt`

```kotlin
package pt.tacho.app.ui

import android.app.Application
import android.content.Intent
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import pt.tacho.app.data.ActivityEntity
import pt.tacho.app.export.Exporter
import pt.tacho.app.repository
import pt.tacho.app.service.TrackingService
import pt.tacho.core.ActivityType
import pt.tacho.core.DriverStatus
import pt.tacho.core.RulesEngine

class TachoViewModel(app: Application) : AndroidViewModel(app) {
    private val repo = app.repository
    private val engine = RulesEngine()
    private val exporter = Exporter(app, repo)

    private val ticker = flow {
        while (true) {
            emit(System.currentTimeMillis())
            delay(1_000)
        }
    }

    /** Estado do condutor, recalculado todos os segundos e sempre que o histórico muda. */
    val status: StateFlow<DriverStatus> = combine(repo.observeRecent(), ticker) { activities, now ->
        engine.evaluate(activities.map { it.toCore() }, now)
    }
        .flowOn(Dispatchers.Default)
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5_000),
            engine.evaluate(emptyList(), System.currentTimeMillis()),
        )

    val history: StateFlow<List<ActivityEntity>> = repo.observeAll()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    fun switchTo(type: ActivityType) {
        viewModelScope.launch {
            repo.switchTo(type)
            TrackingService.sync(getApplication(), type)
        }
    }

    /** Garante que o serviço reflete a atividade atual (chamado quando a app vem para primeiro plano). */
    fun syncService() {
        viewModelScope.launch { TrackingService.sync(getApplication(), repo.currentType()) }
    }

    fun save(entry: ActivityEntity, onResult: (Boolean) -> Unit) {
        viewModelScope.launch {
            val ok = repo.save(entry)
            if (ok) TrackingService.sync(getApplication(), repo.currentType())
            onResult(ok)
        }
    }

    fun delete(entry: ActivityEntity) {
        viewModelScope.launch {
            repo.delete(entry)
            TrackingService.sync(getApplication(), repo.currentType())
        }
    }

    /** Cria o ficheiro e devolve o Intent de partilha, ou null se falhar. */
    suspend fun exportCsv(): Intent? = runCatching { exporter.share(exporter.csv(), "text/csv") }.getOrNull()

    suspend fun exportPdf(): Intent? = runCatching { exporter.share(exporter.pdf(), "application/pdf") }.getOrNull()
}
```

## `app/src/main/java/pt/tacho/app/ui/Theme.kt`

```kotlin
package pt.tacho.app.ui

import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import pt.tacho.core.ActivityType

private val DarkColors = darkColorScheme(
    primary = Color(0xFF8FC1FF),
    onPrimary = Color(0xFF00315C),
    primaryContainer = Color(0xFF1D4A7A),
    background = Color(0xFF10151B),
    surface = Color(0xFF10151B),
    surfaceVariant = Color(0xFF232B35),
    tertiaryContainer = Color(0xFF5A4300),
    onTertiaryContainer = Color(0xFFFFE08A),
)

private val LightColors = lightColorScheme(
    primary = Color(0xFF0B5CAD),
    primaryContainer = Color(0xFFD3E4FF),
    tertiaryContainer = Color(0xFFFFE9A8),
    onTertiaryContainer = Color(0xFF3F2E00),
)

@Composable
fun TachoTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = if (isSystemInDarkTheme()) DarkColors else LightColors,
        content = content,
    )
}

fun activityColor(type: ActivityType): Color = when (type) {
    ActivityType.DRIVING -> Color(0xFF2E7DD7)
    ActivityType.OTHER_WORK -> Color(0xFFE0A100)
    ActivityType.AVAILABILITY -> Color(0xFF8E5BD1)
    ActivityType.REST -> Color(0xFF2E9E5B)
}

val WarningColor = Color(0xFFE0A100)
```

## `app/src/main/java/pt/tacho/app/ui/TodayScreen.kt`

```kotlin
package pt.tacho.app.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Build
import androidx.compose.material.icons.filled.Hotel
import androidx.compose.material.icons.filled.HourglassEmpty
import androidx.compose.material.icons.filled.LocalShipping
import androidx.compose.material3.Button
import androidx.compose.material3.ButtonDefaults
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Icon
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.res.stringResource
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import pt.tacho.app.R
import pt.tacho.app.util.activityLabel
import pt.tacho.app.util.alertMessage
import pt.tacho.app.util.formatClock
import pt.tacho.app.util.formatDayClock
import pt.tacho.app.util.formatHm
import pt.tacho.app.util.formatHms
import pt.tacho.app.util.limitLabel
import pt.tacho.core.ActivityType
import pt.tacho.core.Alert
import pt.tacho.core.AlertLevel
import pt.tacho.core.DriverStatus
import pt.tacho.core.Limits

@Composable
fun TodayScreen(vm: TachoViewModel, modifier: Modifier = Modifier) {
    val status by vm.status.collectAsStateWithLifecycle()

    Column(
        modifier = modifier
            .verticalScroll(rememberScrollState())
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp),
    ) {
        status.alerts.forEach { AlertCard(it) }
        CurrentActivityCard(status)
        ActivityButtons(current = status.currentActivity, onSelect = vm::switchTo)

        if (status.currentActivity != null) {
            CanDriveCard(status)
            BreakCard(status)
            LimitMeter(
                title = stringResource(R.string.meter_continuous),
                used = status.continuousDrivingMs,
                limit = Limits.CONTINUOUS_DRIVING,
            )
            LimitMeter(
                title = stringResource(R.string.meter_daily),
                used = status.dailyDrivingMs,
                limit = status.dailyLimitMs,
                note = stringResource(R.string.daily_extensions, status.extendedDaysThisWeek.coerceAtMost(2)),
            )
            LimitMeter(
                title = stringResource(R.string.meter_weekly),
                used = status.weeklyDrivingMs,
                limit = Limits.WEEKLY_DRIVING,
            )
            LimitMeter(
                title = stringResource(R.string.meter_biweekly),
                used = status.biweeklyDrivingMs,
                limit = Limits.BIWEEKLY_DRIVING,
            )
            DeadlinesCard(status)
        }

        Text(
            text = stringResource(R.string.disclaimer),
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant,
        )
    }
}

@Composable
private fun AlertCard(alert: Alert) {
    val colors = MaterialTheme.colorScheme
    val (container, content) = when (alert.level) {
        AlertLevel.CRITICAL -> colors.errorContainer to colors.onErrorContainer
        AlertLevel.WARNING -> colors.tertiaryContainer to colors.onTertiaryContainer
        AlertLevel.INFO -> colors.secondaryContainer to colors.onSecondaryContainer
    }
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = container, contentColor = content),
    ) {
        Text(
            text = alertMessage(LocalContext.current, alert),
            modifier = Modifier.padding(16.dp),
            style = MaterialTheme.typography.titleMedium,
            fontWeight = FontWeight.SemiBold,
        )
    }
}

@Composable
private fun CurrentActivityCard(status: DriverStatus) {
    val type = status.currentActivity
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            if (type == null) {
                Text(stringResource(R.string.no_activity), style = MaterialTheme.typography.titleMedium)
                Text(
                    stringResource(R.string.choose_activity),
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                )
            } else {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Spacer(
                        Modifier
                            .size(14.dp)
                            .clip(CircleShape)
                            .background(activityColor(type)),
                    )
                    Text(
                        text = stringResource(activityLabel(type)),
                        modifier = Modifier.padding(start = 8.dp),
                        style = MaterialTheme.typography.titleLarge,
                        fontWeight = FontWeight.SemiBold,
                    )
                }
                val since = status.currentActivityStartMs ?: status.nowMs
                Text(
                    text = formatHms(status.nowMs - since),
                    fontSize = 44.sp,
                    fontWeight = FontWeight.Bold,
                )
                Text(
                    text = stringResource(R.string.since, formatClock(since)),
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                )
            }
        }
    }
}

private fun activityIcon(type: ActivityType): ImageVector = when (type) {
    ActivityType.DRIVING -> Icons.Filled.LocalShipping
    ActivityType.OTHER_WORK -> Icons.Filled.Build
    ActivityType.AVAILABILITY -> Icons.Filled.HourglassEmpty
    ActivityType.REST -> Icons.Filled.Hotel
}

@Composable
private fun ActivityButtons(current: ActivityType?, onSelect: (ActivityType) -> Unit) {
    val rows = listOf(
        listOf(ActivityType.DRIVING, ActivityType.OTHER_WORK),
        listOf(ActivityType.AVAILABILITY, ActivityType.REST),
    )
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        rows.forEach { row ->
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                row.forEach { type ->
                    val selected = type == current
                    val color = activityColor(type)
                    Button(
                        onClick = { onSelect(type) },
                        modifier = Modifier
                            .weight(1f)
                            .height(76.dp),
                        shape = RoundedCornerShape(16.dp),
                        colors = ButtonDefaults.buttonColors(
                            containerColor = if (selected) color else color.copy(alpha = 0.22f),
                            contentColor = if (selected) Color.White else MaterialTheme.colorScheme.onSurface,
                        ),
                    ) {
                        Icon(activityIcon(type), contentDescription = null)
                        Text(
                            text = stringResource(activityLabel(type)),
                            modifier = Modifier.padding(start = 8.dp),
                            fontWeight = FontWeight.SemiBold,
                        )
                    }
                }
            }
        }
    }
}

@Composable
private fun CanDriveCard(status: DriverStatus) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.primaryContainer),
    ) {
        Column(Modifier.padding(16.dp)) {
            Text(stringResource(R.string.can_drive_title), style = MaterialTheme.typography.titleMedium)
            Text(formatHm(status.canDriveMs), fontSize = 40.sp, fontWeight = FontWeight.Bold)
            Text(
                stringResource(R.string.limited_by, stringResource(limitLabel(status.limitingFactor))),
                style = MaterialTheme.typography.bodyMedium,
            )
        }
    }
}

@Composable
private fun BreakCard(status: DriverStatus) {
    if (status.breakNeededMs <= 0) return
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            Text(stringResource(R.string.break_title), style = MaterialTheme.typography.titleMedium)
            val text = if (status.breakRemainingMs > 0) {
                stringResource(
                    R.string.break_progress,
                    formatHm(status.currentRestMs),
                    formatHm(status.breakRemainingMs),
                )
            } else {
                stringResource(R.string.break_done)
            }
            Text(text, style = MaterialTheme.typography.bodyMedium)
        }
    }
}

@Composable
private fun LimitMeter(title: String, used: Long, limit: Long, note: String? = null) {
    val remaining = limit - used
    val fraction = (used.toFloat() / limit).coerceIn(0f, 1f)
    val barColor = when {
        remaining <= 0 -> MaterialTheme.colorScheme.error
        fraction >= 0.9f -> WarningColor
        else -> MaterialTheme.colorScheme.primary
    }
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
                Text(title, style = MaterialTheme.typography.titleSmall)
                Text(
                    stringResource(R.string.meter_used_of, formatHm(used), formatHm(limit)),
                    style = MaterialTheme.typography.titleSmall,
                    fontWeight = FontWeight.SemiBold,
                )
            }
            LinearProgressIndicator(
                progress = { fraction },
                modifier = Modifier
                    .fillMaxWidth()
                    .height(10.dp)
                    .clip(RoundedCornerShape(5.dp)),
                color = barColor,
            )
            Text(
                text = if (remaining > 0) stringResource(R.string.meter_remaining, formatHm(remaining))
                else stringResource(R.string.meter_exceeded, formatHm(-remaining)),
                style = MaterialTheme.typography.bodySmall,
                color = if (remaining > 0) MaterialTheme.colorScheme.onSurfaceVariant else MaterialTheme.colorScheme.error,
            )
            if (note != null) {
                Text(note, style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
        }
    }
}

@Composable
private fun DeadlinesCard(status: DriverStatus) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
            Text(stringResource(R.string.deadlines_title), style = MaterialTheme.typography.titleMedium)
            val regularOnly = status.reducedDailyRestsUsed >= Limits.MAX_REDUCED_DAILY_RESTS
            Text(
                text = if (regularOnly) {
                    stringResource(R.string.deadline_daily_regular_only, formatDayClock(status.dailyRestRegularStartByMs))
                } else {
                    stringResource(
                        R.string.deadline_daily,
                        formatDayClock(status.dailyRestRegularStartByMs),
                        formatDayClock(status.dailyRestReducedStartByMs),
                    )
                },
                style = MaterialTheme.typography.bodyMedium,
            )
            Text(
                stringResource(R.string.reduced_rests_used, status.reducedDailyRestsUsed),
                style = MaterialTheme.typography.bodyMedium,
            )
            Text(
                stringResource(R.string.deadline_weekly, formatDayClock(status.weeklyRestStartByMs)),
                style = MaterialTheme.typography.bodyMedium,
            )
            if (status.weeklyAnchorAssumed) {
                Text(
                    stringResource(R.string.weekly_anchor_assumed),
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                )
            }
        }
    }
}
```

## `app/src/main/java/pt/tacho/app/util/Format.kt`

```kotlin
package pt.tacho.app.util

import android.content.Context
import androidx.annotation.StringRes
import pt.tacho.app.R
import pt.tacho.core.ActivityType
import pt.tacho.core.Alert
import pt.tacho.core.AlertCode
import pt.tacho.core.AlertLevel
import pt.tacho.core.LimitKind
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneId
import java.time.format.DateTimeFormatter
import java.util.Locale
import kotlin.math.abs

private val PT: Locale = Locale.forLanguageTag("pt-PT")
private val CLOCK = DateTimeFormatter.ofPattern("HH:mm", PT)
private val DAY_CLOCK = DateTimeFormatter.ofPattern("EEE dd/MM HH:mm", PT)
private val DATE_TIME = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm", PT)
private val DAY_HEADER = DateTimeFormatter.ofPattern("EEEE, d 'de' MMMM", PT)
private val DATE_SHORT = DateTimeFormatter.ofPattern("dd/MM/yyyy", PT)

/** "H:MM" (com sinal negativo se for o caso). */
fun formatHm(ms: Long): String {
    val totalMin = abs(ms) / 60_000
    val sign = if (ms < 0) "-" else ""
    return "$sign${totalMin / 60}:${(totalMin % 60).toString().padStart(2, '0')}"
}

/** "H:MM:SS". */
fun formatHms(ms: Long): String {
    val totalSec = abs(ms) / 1000
    return "${totalSec / 3600}:${((totalSec / 60) % 60).toString().padStart(2, '0')}:${(totalSec % 60).toString().padStart(2, '0')}"
}

private fun zoned(ms: Long) = Instant.ofEpochMilli(ms).atZone(ZoneId.systemDefault())

fun formatClock(ms: Long): String = CLOCK.format(zoned(ms))
fun formatDayClock(ms: Long): String = DAY_CLOCK.format(zoned(ms))
fun formatDateTime(ms: Long): String = DATE_TIME.format(zoned(ms))
fun formatDayHeader(date: LocalDate): String = DAY_HEADER.format(date).replaceFirstChar { it.titlecase(PT) }
fun formatDate(date: LocalDate): String = DATE_SHORT.format(date)
fun formatDateTimeNow(ms: Long): String = formatDateTime(ms)

@StringRes
fun activityLabel(type: ActivityType): Int = when (type) {
    ActivityType.DRIVING -> R.string.activity_driving
    ActivityType.OTHER_WORK -> R.string.activity_other_work
    ActivityType.AVAILABILITY -> R.string.activity_availability
    ActivityType.REST -> R.string.activity_rest
}

@StringRes
fun limitLabel(kind: LimitKind): Int = when (kind) {
    LimitKind.CONTINUOUS -> R.string.limit_continuous
    LimitKind.DAILY -> R.string.limit_daily
    LimitKind.WEEKLY -> R.string.limit_weekly
    LimitKind.BIWEEKLY -> R.string.limit_biweekly
}

fun alertTitle(context: Context, level: AlertLevel): String = context.getString(
    when (level) {
        AlertLevel.INFO -> R.string.alert_title_info
        AlertLevel.WARNING -> R.string.alert_title_warning
        AlertLevel.CRITICAL -> R.string.alert_title_critical
    },
)

fun alertMessage(context: Context, alert: Alert): String {
    val exceeded = alert.remainingMs <= 0
    val res = when (alert.code) {
        AlertCode.CONTINUOUS_DRIVING ->
            if (exceeded) R.string.alert_continuous_exceeded else R.string.alert_continuous_remaining
        AlertCode.DAILY_DRIVING ->
            if (exceeded) R.string.alert_daily_exceeded else R.string.alert_daily_remaining
        AlertCode.WEEKLY_DRIVING ->
            if (exceeded) R.string.alert_weekly_exceeded else R.string.alert_weekly_remaining
        AlertCode.BIWEEKLY_DRIVING ->
            if (exceeded) R.string.alert_biweekly_exceeded else R.string.alert_biweekly_remaining
        AlertCode.DAILY_REST_DEADLINE ->
            if (exceeded) R.string.alert_daily_rest_exceeded else R.string.alert_daily_rest_remaining
        AlertCode.WEEKLY_REST_DEADLINE ->
            if (exceeded) R.string.alert_weekly_rest_exceeded else R.string.alert_weekly_rest_remaining
    }
    return context.getString(res, formatHm(abs(alert.remainingMs)))
}
```

## `app/src/main/res/drawable/ic_launcher_foreground.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <!-- Relógio simples: aro + ponteiros -->
    <path
        android:fillColor="#00000000"
        android:pathData="M54,54m-24,0a24,24 0,1 1,48 0a24,24 0,1 1,-48 0"
        android:strokeColor="#FFC83D"
        android:strokeWidth="5" />
    <path
        android:fillColor="#00000000"
        android:pathData="M54,38L54,54L65,61"
        android:strokeColor="#FFFFFF"
        android:strokeLineCap="round"
        android:strokeLineJoin="round"
        android:strokeWidth="5" />
</vector>
```

## `app/src/main/res/drawable/ic_notification.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#FFFFFFFF"
        android:pathData="M12,2C6.5,2 2,6.5 2,12s4.5,10 10,10 10,-4.5 10,-10S17.5,2 12,2zM12,20c-4.4,0 -8,-3.6 -8,-8s3.6,-8 8,-8 8,3.6 8,8 -3.6,8 -8,8zM12.5,7H11v6l5.2,3.2 0.8,-1.3 -4.5,-2.7V7z" />
</vector>
```

## `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
```

## `app/src/main/res/values/colors.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="ic_launcher_background">#0F2A43</color>
</resources>
```

## `app/src/main/res/values/strings.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Tempos de Condução</string>
    <string name="tab_today">Hoje</string>
    <string name="tab_history">Histórico</string>

    <string name="activity_driving">Condução</string>
    <string name="activity_other_work">Outro trabalho</string>
    <string name="activity_availability">Disponibilidade</string>
    <string name="activity_rest">Descanso</string>

    <string name="no_activity">Sem atividade registada</string>
    <string name="choose_activity">Toque numa atividade para começar</string>
    <string name="since">desde %1$s</string>

    <string name="can_drive_title">Pode conduzir ainda</string>
    <string name="limited_by">Limite mais próximo: %1$s</string>
    <string name="limit_continuous">pausa obrigatória</string>
    <string name="limit_daily">condução diária</string>
    <string name="limit_weekly">condução semanal</string>
    <string name="limit_biweekly">condução quinzenal</string>

    <string name="meter_continuous">Condução contínua</string>
    <string name="meter_daily">Condução diária</string>
    <string name="meter_weekly">Condução semanal</string>
    <string name="meter_biweekly">Condução quinzenal</string>
    <string name="meter_used_of">%1$s de %2$s</string>
    <string name="meter_remaining">faltam %1$s</string>
    <string name="meter_exceeded">excedido em %1$s</string>
    <string name="daily_extensions">Dias de 10h esta semana: %1$d de 2</string>

    <string name="break_title">Pausa em curso</string>
    <string name="break_progress">%1$s de pausa · faltam %2$s para repor a condução contínua</string>
    <string name="break_done">Pausa concluída: condução contínua reposta</string>

    <string name="deadlines_title">Prazos de descanso</string>
    <string name="deadline_daily">Descanso diário: iniciar até %1$s (11h) ou %2$s (9h)</string>
    <string name="deadline_daily_regular_only">Descanso diário: iniciar até %1$s (tem de ser de 11h)</string>
    <string name="reduced_rests_used">Descansos diários reduzidos usados: %1$d de 3</string>
    <string name="deadline_weekly">Descanso semanal: iniciar até %1$s</string>
    <string name="weekly_anchor_assumed">Ainda não há descanso semanal registado: o prazo conta desde o primeiro registo.</string>

    <string name="alert_title_info">Atenção</string>
    <string name="alert_title_warning">Aviso</string>
    <string name="alert_title_critical">Limite atingido</string>
    <string name="alert_continuous_remaining">Faltam %1$s para a pausa obrigatória</string>
    <string name="alert_continuous_exceeded">Pausa obrigatória em atraso há %1$s</string>
    <string name="alert_daily_remaining">Faltam %1$s para o limite de condução diária</string>
    <string name="alert_daily_exceeded">Limite de condução diária ultrapassado há %1$s</string>
    <string name="alert_weekly_remaining">Faltam %1$s para o limite semanal de 56h</string>
    <string name="alert_weekly_exceeded">Limite semanal ultrapassado há %1$s</string>
    <string name="alert_biweekly_remaining">Faltam %1$s para o limite quinzenal de 90h</string>
    <string name="alert_biweekly_exceeded">Limite quinzenal ultrapassado há %1$s</string>
    <string name="alert_daily_rest_remaining">Faltam %1$s para iniciar o descanso diário</string>
    <string name="alert_daily_rest_exceeded">Descanso diário em atraso há %1$s</string>
    <string name="alert_weekly_rest_remaining">Faltam %1$s para iniciar o descanso semanal</string>
    <string name="alert_weekly_rest_exceeded">Descanso semanal em atraso há %1$s</string>

    <string name="history_empty">Ainda não há registos.</string>
    <string name="add_entry">Adicionar</string>
    <string name="new_entry">Novo registo</string>
    <string name="edit_entry">Editar registo</string>
    <string name="entry_type">Tipo</string>
    <string name="entry_start">Início</string>
    <string name="entry_end">Fim</string>
    <string name="entry_ongoing">em curso</string>
    <string name="save">Guardar</string>
    <string name="cancel">Cancelar</string>
    <string name="delete">Apagar</string>
    <string name="error_range">O fim tem de ser depois do início e não pode estar no futuro.</string>
    <string name="export_csv">CSV</string>
    <string name="export_pdf">PDF</string>
    <string name="export_share">Partilhar relatório</string>
    <string name="export_failed">Não foi possível criar o relatório.</string>
    <string name="day_totals">Condução %1$s · Outro trabalho %2$s</string>

    <string name="channel_ongoing">Registo em curso</string>
    <string name="channel_ongoing_desc">Mostra a atividade atual e quanto tempo pode ainda conduzir.</string>
    <string name="channel_alerts">Alertas de limites</string>
    <string name="channel_alerts_desc">Avisa antes de atingir os limites de condução e descanso.</string>
    <string name="notif_ongoing_placeholder">A registar tempos…</string>
    <string name="notif_ongoing_text">Pode conduzir ainda %1$s</string>

    <string name="report_title">Relatório de tempos de condução e descanso</string>
    <string name="report_generated">Gerado em %1$s</string>
    <string name="report_col_date">Data</string>
    <string name="report_col_driving">Condução</string>
    <string name="report_col_work">Outro trab.</string>
    <string name="report_col_availability">Disponib.</string>
    <string name="report_col_rest">Descanso reg.</string>
    <string name="report_weeks_title">Condução por semana (limite 56h)</string>
    <string name="report_week_of">Semana de %1$s</string>
    <string name="report_disclaimer">Documento de apoio ao condutor. Não substitui os registos do tacógrafo.</string>

    <string name="disclaimer">Esta aplicação é uma ajuda ao condutor e não substitui o tacógrafo nem tem valor legal. Baseia-se no Regulamento (CE) n.º 561/2006 e não cobre todas as exceções. Os intervalos sem registo contam como descanso.</string>
</resources>
```

## `app/src/main/res/values/themes.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="Theme.Tacho" parent="android:Theme.Material.NoActionBar">
        <item name="android:statusBarColor">@android:color/transparent</item>
        <item name="android:navigationBarColor">@android:color/transparent</item>
    </style>
</resources>
```

## `app/src/main/res/xml/file_paths.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="exports" path="exports/" />
</paths>
```

## `build.gradle.kts`

```kotlin
plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
    id("org.jetbrains.kotlin.jvm") version "2.0.21" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
    id("com.google.devtools.ksp") version "2.0.21-1.0.28" apply false
}
```

## `core/build.gradle.kts`

```kotlin
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

plugins {
    id("org.jetbrains.kotlin.jvm")
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

kotlin {
    compilerOptions {
        jvmTarget.set(JvmTarget.JVM_17)
    }
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}
```

## `core/src/main/kotlin/pt/tacho/core/Models.kt`

```kotlin
package pt.tacho.core

/** Atividades registadas pelo condutor (equivalentes aos símbolos do tacógrafo). */
enum class ActivityType { DRIVING, OTHER_WORK, AVAILABILITY, REST }

/** Uma atividade; [endMs] nulo significa "em curso". Tempos em milissegundos desde a época (UTC). */
data class Activity(
    val type: ActivityType,
    val startMs: Long,
    val endMs: Long? = null,
)

/** Limites do Regulamento (CE) n.º 561/2006. */
object Limits {
    const val MINUTE = 60_000L
    const val HOUR = 60 * MINUTE
    const val DAY = 24 * HOUR

    // Condução contínua e pausas (art. 7.º)
    const val CONTINUOUS_DRIVING = 4 * HOUR + 30 * MINUTE
    const val BREAK_FULL = 45 * MINUTE
    const val BREAK_FIRST_PART = 15 * MINUTE   // 1.ª parte (≥15) ...
    const val BREAK_SECOND_PART = 30 * MINUTE  // ... seguida de 2.ª parte (≥30)

    // Condução diária, semanal e quinzenal (art. 6.º)
    const val DAILY_DRIVING = 9 * HOUR
    const val DAILY_DRIVING_EXTENDED = 10 * HOUR
    const val MAX_EXTENDED_DAYS_PER_WEEK = 2
    const val WEEKLY_DRIVING = 56 * HOUR
    const val BIWEEKLY_DRIVING = 90 * HOUR

    // Descansos (art. 8.º)
    const val DAILY_REST_REGULAR = 11 * HOUR
    const val DAILY_REST_REDUCED = 9 * HOUR
    const val DAILY_REST_SPLIT_FIRST = 3 * HOUR // regular fracionado: 3h + 9h
    const val MAX_REDUCED_DAILY_RESTS = 3       // entre dois descansos semanais
    const val WEEKLY_REST_REGULAR = 45 * HOUR
    const val WEEKLY_REST_REDUCED = 24 * HOUR
    const val WEEKLY_REST_PERIODS = 6           // descanso semanal até ao fim de 6 períodos de 24h
}

enum class LimitKind { CONTINUOUS, DAILY, WEEKLY, BIWEEKLY }

enum class AlertLevel { INFO, WARNING, CRITICAL }

enum class AlertCode {
    CONTINUOUS_DRIVING,
    DAILY_DRIVING,
    WEEKLY_DRIVING,
    BIWEEKLY_DRIVING,
    DAILY_REST_DEADLINE,
    WEEKLY_REST_DEADLINE,
}

/** [remainingMs] negativo = limite ultrapassado há esse tempo. */
data class Alert(
    val code: AlertCode,
    val level: AlertLevel,
    val remainingMs: Long,
)

data class DriverStatus(
    val nowMs: Long,
    val currentActivity: ActivityType?,
    val currentActivityStartMs: Long?,

    // Condução contínua / pausa
    val continuousDrivingMs: Long,
    val continuousRemainingMs: Long,
    /** Duração mínima da pausa em curso para repor a condução contínua (0 se não aplicável). */
    val breakNeededMs: Long,
    val breakRemainingMs: Long,
    val currentRestMs: Long,

    // Condução diária
    val dailyDrivingMs: Long,
    val dailyLimitMs: Long,
    val dailyRemainingMs: Long,
    val extendedDaysThisWeek: Int,

    // Condução semanal e quinzenal (semana fixa: segunda 00:00 – domingo 24:00)
    val weeklyDrivingMs: Long,
    val weeklyRemainingMs: Long,
    val biweeklyDrivingMs: Long,
    val biweeklyRemainingMs: Long,

    // Resumo: quanto ainda pode conduzir e qual o limite que primeiro se atinge
    val canDriveMs: Long,
    val limitingFactor: LimitKind,

    // Descansos
    val reducedDailyRestsUsed: Int,
    /** Hora limite para INICIAR o descanso diário de 11h / de 9h (reduzido). */
    val dailyRestRegularStartByMs: Long,
    val dailyRestReducedStartByMs: Long,
    /** Hora limite para INICIAR o descanso semanal. */
    val weeklyRestStartByMs: Long,
    /** true se não há nenhum descanso semanal registado: prazo calculado desde o primeiro registo. */
    val weeklyAnchorAssumed: Boolean,

    val alerts: List<Alert>,
)
```

## `core/src/main/kotlin/pt/tacho/core/RulesEngine.kt`

```kotlin
package pt.tacho.core

import pt.tacho.core.Limits.BIWEEKLY_DRIVING
import pt.tacho.core.Limits.BREAK_FIRST_PART
import pt.tacho.core.Limits.BREAK_FULL
import pt.tacho.core.Limits.BREAK_SECOND_PART
import pt.tacho.core.Limits.CONTINUOUS_DRIVING
import pt.tacho.core.Limits.DAILY_DRIVING
import pt.tacho.core.Limits.DAILY_DRIVING_EXTENDED
import pt.tacho.core.Limits.DAILY_REST_REDUCED
import pt.tacho.core.Limits.DAILY_REST_REGULAR
import pt.tacho.core.Limits.DAILY_REST_SPLIT_FIRST
import pt.tacho.core.Limits.DAY
import pt.tacho.core.Limits.HOUR
import pt.tacho.core.Limits.MAX_EXTENDED_DAYS_PER_WEEK
import pt.tacho.core.Limits.MAX_REDUCED_DAILY_RESTS
import pt.tacho.core.Limits.MINUTE
import pt.tacho.core.Limits.WEEKLY_DRIVING
import pt.tacho.core.Limits.WEEKLY_REST_PERIODS
import pt.tacho.core.Limits.WEEKLY_REST_REDUCED
import java.time.DayOfWeek
import java.time.Instant
import java.time.ZoneId
import java.time.ZoneOffset
import java.time.temporal.TemporalAdjusters

/**
 * Motor de regras do Regulamento (CE) n.º 561/2006. É uma função pura: recebe o histórico de
 * atividades e o instante atual e devolve o estado do condutor. Não depende de Android.
 *
 * Pressupostos (documentados de propósito):
 *  - Intervalos sem registo contam como descanso; o tempo depois da última atividade fechada também.
 *  - Só a atividade REST conta como pausa/descanso (outro trabalho e disponibilidade não).
 *  - A semana é fixa (segunda 00:00 a domingo 24:00) em [weekZone], por omissão UTC como no tacógrafo.
 *  - Se não houver descanso registado, os prazos contam desde o primeiro registo.
 *  - A compensação de descansos semanais reduzidos ainda não é acompanhada.
 *  - Exceções (transporte ferry/comboio, 2 condutores, transporte internacional de passageiros,
 *    condução fora do âmbito do regulamento) não estão modeladas.
 */
class RulesEngine(private val weekZone: ZoneId = ZoneOffset.UTC) {

    internal data class Seg(val type: ActivityType, val start: Long, val end: Long) {
        val duration get() = end - start
    }

    fun evaluate(activities: List<Activity>, nowMs: Long): DriverStatus {
        val timeline = buildTimeline(activities, nowMs)
        val anchor = timeline.firstOrNull()?.start ?: nowMs

        var driveSinceBreak = 0L
        var hasFirstPart = false          // já houve uma interrupção ≥15 min (1.ª parte de 15+30)
        var dayDriving = 0L
        var dayStart = anchor
        var dayHadSplitPart = false       // já houve um repouso ≥3h neste dia (descanso fracionado 3+9)
        var reducedUsed = 0
        var lastDayRestEnd = anchor
        var lastWeeklyRestEnd = anchor
        var weeklyAnchorAssumed = true
        val extendedDayStarts = ArrayList<Long>()

        var preDrive = 0L
        var preFirstPart = false
        var currentRest = 0L

        timeline.forEachIndexed { i, seg ->
            val dur = seg.duration
            val isLast = i == timeline.lastIndex
            when (seg.type) {
                ActivityType.DRIVING -> {
                    driveSinceBreak += dur
                    dayDriving += dur
                }

                ActivityType.OTHER_WORK, ActivityType.AVAILABILITY -> Unit

                ActivityType.REST -> {
                    if (isLast) {
                        preDrive = driveSinceBreak
                        preFirstPart = hasFirstPart
                        currentRest = dur
                    }

                    // Pausa (art. 7.º): 45 min, ou 15 min seguidos de 30 min.
                    when {
                        dur >= BREAK_FULL -> {
                            driveSinceBreak = 0; hasFirstPart = false
                        }
                        hasFirstPart && dur >= BREAK_SECOND_PART -> {
                            driveSinceBreak = 0; hasFirstPart = false
                        }
                        dur >= BREAK_FIRST_PART -> hasFirstPart = true
                    }

                    // Descanso diário/semanal (art. 8.º)
                    if (dur >= DAILY_REST_REDUCED) {
                        if (dayDriving > DAILY_DRIVING) extendedDayStarts += dayStart
                        if (dur >= WEEKLY_REST_REDUCED) {
                            reducedUsed = 0
                            lastWeeklyRestEnd = seg.end
                            weeklyAnchorAssumed = false
                        } else if (dur < DAILY_REST_REGULAR && !dayHadSplitPart && !isLast) {
                            // Só conta como reduzido quando já terminou (o último pode ainda crescer até 11h).
                            reducedUsed++
                        }
                        dayDriving = 0
                        dayStart = seg.end
                        dayHadSplitPart = false
                        lastDayRestEnd = seg.end
                    } else if (dur >= DAILY_REST_SPLIT_FIRST) {
                        dayHadSplitPart = true
                    }
                }
            }
        }

        val current = timeline.lastOrNull()
        val currentType = current?.type
        val isResting = currentType == ActivityType.REST

        // Semana fixa e quinzena
        val weekStart = weekStartMs(nowMs, 0)
        val weekEnd = weekStartMs(nowMs, 1)
        val prevWeekStart = weekStartMs(nowMs, -1)
        val driving = timeline.filter { it.type == ActivityType.DRIVING }
        val weeklyDriving = drivingWithin(driving, weekStart, weekEnd)
        val biweeklyDriving = weeklyDriving + drivingWithin(driving, prevWeekStart, weekStart)

        // Prolongamentos a 10h: no máximo 2 por semana
        val priorExtended = extendedDayStarts.count { it in weekStart until weekEnd }
        val dailyLimit =
            if (priorExtended < MAX_EXTENDED_DAYS_PER_WEEK) DAILY_DRIVING_EXTENDED else DAILY_DRIVING
        val extendedThisWeek = priorExtended + if (dayDriving > DAILY_DRIVING) 1 else 0

        val continuousRemaining = CONTINUOUS_DRIVING - driveSinceBreak
        val dailyRemaining = dailyLimit - dayDriving
        val weeklyRemaining = WEEKLY_DRIVING - weeklyDriving
        val biweeklyRemaining = BIWEEKLY_DRIVING - biweeklyDriving

        val limits = listOf(
            LimitKind.CONTINUOUS to continuousRemaining,
            LimitKind.DAILY to dailyRemaining,
            LimitKind.WEEKLY to weeklyRemaining,
            LimitKind.BIWEEKLY to biweeklyRemaining,
        )
        val limiting = limits.minByOrNull { it.second }!!

        // Pausa em curso
        val breakNeeded = if (isResting && preDrive > 0) {
            if (preFirstPart) BREAK_SECOND_PART else BREAK_FULL
        } else 0L
        val breakRemaining = if (breakNeeded > 0) maxOf(0L, breakNeeded - currentRest) else 0L

        // Prazos
        val regularBy = lastDayRestEnd + DAY - DAILY_REST_REGULAR
        val reducedBy = lastDayRestEnd + DAY - DAILY_REST_REDUCED
        val weeklyBy = lastWeeklyRestEnd + WEEKLY_REST_PERIODS * DAY
        val dailyRestMustStartBy = if (reducedUsed < MAX_REDUCED_DAILY_RESTS) reducedBy else regularBy

        val alerts = buildList {
            if (currentType == ActivityType.DRIVING) {
                levelFor(continuousRemaining, 15 * MINUTE, 30 * MINUTE)
                    ?.let { add(Alert(AlertCode.CONTINUOUS_DRIVING, it, continuousRemaining)) }
                levelFor(dailyRemaining, 30 * MINUTE, 60 * MINUTE)
                    ?.let { add(Alert(AlertCode.DAILY_DRIVING, it, dailyRemaining)) }
                levelFor(weeklyRemaining, HOUR, 3 * HOUR)
                    ?.let { add(Alert(AlertCode.WEEKLY_DRIVING, it, weeklyRemaining)) }
                levelFor(biweeklyRemaining, HOUR, 3 * HOUR)
                    ?.let { add(Alert(AlertCode.BIWEEKLY_DRIVING, it, biweeklyRemaining)) }
            }
            if (current != null && !isResting) {
                val dailyRestIn = dailyRestMustStartBy - nowMs
                levelFor(dailyRestIn, 30 * MINUTE, 60 * MINUTE)
                    ?.let { add(Alert(AlertCode.DAILY_REST_DEADLINE, it, dailyRestIn)) }
                val weeklyRestIn = weeklyBy - nowMs
                levelFor(weeklyRestIn, 3 * HOUR, 12 * HOUR)
                    ?.let { add(Alert(AlertCode.WEEKLY_REST_DEADLINE, it, weeklyRestIn)) }
            }
        }.sortedWith(compareByDescending<Alert> { it.level }.thenBy { it.remainingMs })

        return DriverStatus(
            nowMs = nowMs,
            currentActivity = currentType,
            currentActivityStartMs = current?.start,
            continuousDrivingMs = driveSinceBreak,
            continuousRemainingMs = continuousRemaining,
            breakNeededMs = breakNeeded,
            breakRemainingMs = breakRemaining,
            currentRestMs = if (isResting) currentRest else 0L,
            dailyDrivingMs = dayDriving,
            dailyLimitMs = dailyLimit,
            dailyRemainingMs = dailyRemaining,
            extendedDaysThisWeek = extendedThisWeek,
            weeklyDrivingMs = weeklyDriving,
            weeklyRemainingMs = weeklyRemaining,
            biweeklyDrivingMs = biweeklyDriving,
            biweeklyRemainingMs = biweeklyRemaining,
            canDriveMs = maxOf(0L, limiting.second),
            limitingFactor = limiting.first,
            reducedDailyRestsUsed = reducedUsed,
            dailyRestRegularStartByMs = regularBy,
            dailyRestReducedStartByMs = reducedBy,
            weeklyRestStartByMs = weeklyBy,
            weeklyAnchorAssumed = weeklyAnchorAssumed,
            alerts = alerts,
        )
    }

    private fun levelFor(remaining: Long, warnAt: Long, infoAt: Long): AlertLevel? = when {
        remaining <= 0 -> AlertLevel.CRITICAL
        remaining <= warnAt -> AlertLevel.WARNING
        remaining <= infoAt -> AlertLevel.INFO
        else -> null
    }

    private fun drivingWithin(driving: List<Seg>, from: Long, to: Long): Long =
        driving.sumOf { maxOf(0L, minOf(it.end, to) - maxOf(it.start, from)) }

    private fun weekStartMs(nowMs: Long, offsetWeeks: Long): Long {
        val date = Instant.ofEpochMilli(nowMs).atZone(weekZone).toLocalDate()
        val monday = date.with(TemporalAdjusters.previousOrSame(DayOfWeek.MONDAY)).plusWeeks(offsetWeeks)
        return monday.atStartOfDay(weekZone).toInstant().toEpochMilli()
    }

    /**
     * Normaliza o histórico: ordena, corta ao instante atual, resolve sobreposições (a atividade
     * mais recente prevalece), preenche lacunas com descanso e junta descansos consecutivos.
     */
    internal fun buildTimeline(activities: List<Activity>, nowMs: Long): List<Seg> {
        val sorted = activities
            .filter { it.startMs < nowMs }
            .map { Seg(it.type, it.startMs, minOf(it.endMs ?: nowMs, nowMs)) }
            .filter { it.end > it.start }
            .sortedWith(compareBy<Seg> { it.start }.thenBy { it.end })

        val out = ArrayList<Seg>()
        for (s in sorted) {
            val last = out.lastOrNull()
            if (last != null) {
                when {
                    s.start == last.start -> out.removeAt(out.lastIndex)
                    s.start < last.end -> out[out.lastIndex] = last.copy(end = s.start)
                    s.start > last.end -> out.add(Seg(ActivityType.REST, last.end, s.start))
                }
            }
            out.add(s)
        }
        out.lastOrNull()?.let { if (it.end < nowMs) out.add(Seg(ActivityType.REST, it.end, nowMs)) }

        val merged = ArrayList<Seg>()
        for (s in out) {
            val prev = merged.lastOrNull()
            if (prev != null && prev.type == ActivityType.REST && s.type == ActivityType.REST && prev.end == s.start) {
                merged[merged.lastIndex] = prev.copy(end = s.end)
            } else merged.add(s)
        }
        return merged
    }
}
```

## `core/src/main/kotlin/pt/tacho/core/Summaries.kt`

```kotlin
package pt.tacho.core

import java.time.DayOfWeek
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneId
import java.time.temporal.TemporalAdjusters
import java.util.SortedMap
import java.util.TreeMap

/** Totais registados num dia civil (atividades cortadas à meia-noite do fuso [zone]). */
data class DaySummary(
    val date: LocalDate,
    val drivingMs: Long,
    val otherWorkMs: Long,
    val availabilityMs: Long,
    /** Só o descanso explicitamente registado (não inclui lacunas sem registo). */
    val restMs: Long,
)

/** Resumo por dia, por ordem cronológica crescente. */
fun summarizeByDay(activities: List<Activity>, zone: ZoneId, nowMs: Long): List<DaySummary> {
    val acc: SortedMap<LocalDate, LongArray> = TreeMap()
    for (a in activities) {
        val end = minOf(a.endMs ?: nowMs, nowMs)
        var cursor = a.startMs
        if (end <= cursor) continue
        val idx = when (a.type) {
            ActivityType.DRIVING -> 0
            ActivityType.OTHER_WORK -> 1
            ActivityType.AVAILABILITY -> 2
            ActivityType.REST -> 3
        }
        while (cursor < end) {
            val date = Instant.ofEpochMilli(cursor).atZone(zone).toLocalDate()
            val dayEnd = date.plusDays(1).atStartOfDay(zone).toInstant().toEpochMilli()
            val chunkEnd = minOf(end, dayEnd)
            acc.getOrPut(date) { LongArray(4) }[idx] += chunkEnd - cursor
            cursor = chunkEnd
        }
    }
    return acc.map { (date, v) -> DaySummary(date, v[0], v[1], v[2], v[3]) }
}

/** Condução total por semana fixa (segunda a domingo em [zone]); a chave é a segunda-feira. */
fun summarizeDrivingByWeek(activities: List<Activity>, zone: ZoneId, nowMs: Long): List<Pair<LocalDate, Long>> =
    summarizeByDay(activities, zone, nowMs)
        .groupBy { it.date.with(TemporalAdjusters.previousOrSame(DayOfWeek.MONDAY)) }
        .map { (monday, days) -> monday to days.sumOf { it.drivingMs } }
        .sortedBy { it.first }
```

## `core/src/test/kotlin/pt/tacho/core/RulesEngineTest.kt`

```kotlin
package pt.tacho.core

import pt.tacho.core.ActivityType.AVAILABILITY
import pt.tacho.core.ActivityType.DRIVING
import pt.tacho.core.ActivityType.OTHER_WORK
import pt.tacho.core.ActivityType.REST
import java.time.Instant
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertFalse
import kotlin.test.assertNotNull
import kotlin.test.assertNull
import kotlin.test.assertTrue

private const val H = Limits.HOUR
private const val M = Limits.MINUTE

/** Segunda-feira 2026-10-19 00:00 UTC. */
private val MONDAY = Instant.parse("2026-10-19T00:00:00Z").toEpochMilli()

/** Constrói uma linha temporal contínua a partir de [start]. */
private class Timeline(start: Long = MONDAY) {
    private var cursor = start
    val activities = mutableListOf<Activity>()

    fun add(type: ActivityType, ms: Long) = apply {
        activities += Activity(type, cursor, cursor + ms)
        cursor += ms
    }

    fun drive(ms: Long) = add(DRIVING, ms)
    fun rest(ms: Long) = add(REST, ms)
    fun work(ms: Long) = add(OTHER_WORK, ms)
    fun skip(ms: Long) = apply { cursor += ms }
    val end get() = cursor
}

class RulesEngineTest {
    private val engine = RulesEngine()

    private fun status(t: Timeline, now: Long = t.end) = engine.evaluate(t.activities, now)

    // ---------- Condução contínua e pausas ----------

    @Test
    fun `condução contínua conta até 4h30 e gera alerta crítico`() {
        val s = status(Timeline().drive(4 * H + 30 * M))
        assertEquals(4 * H + 30 * M, s.continuousDrivingMs)
        assertEquals(0, s.continuousRemainingMs)
        assertEquals(LimitKind.CONTINUOUS, s.limitingFactor)
        assertEquals(0, s.canDriveMs)
        val alert = s.alerts.first { it.code == AlertCode.CONTINUOUS_DRIVING }
        assertEquals(AlertLevel.CRITICAL, alert.level)
    }

    @Test
    fun `alerta de aviso a 15 min e informativo a 30 min do limite contínuo`() {
        val warn = status(Timeline().drive(4 * H + 20 * M))
        assertEquals(AlertLevel.WARNING, warn.alerts.first { it.code == AlertCode.CONTINUOUS_DRIVING }.level)
        val info = status(Timeline().drive(4 * H))
        assertEquals(AlertLevel.INFO, info.alerts.first { it.code == AlertCode.CONTINUOUS_DRIVING }.level)
        val none = status(Timeline().drive(3 * H))
        assertTrue(none.alerts.none { it.code == AlertCode.CONTINUOUS_DRIVING })
    }

    @Test
    fun `pausa de 45 min repõe a condução contínua`() {
        val s = status(Timeline().drive(4 * H).rest(45 * M).drive(1 * H))
        assertEquals(1 * H, s.continuousDrivingMs)
        assertEquals(5 * H, s.dailyDrivingMs)
    }

    @Test
    fun `pausa de 44 min não repõe`() {
        val s = status(Timeline().drive(4 * H).rest(44 * M).drive(1 * H))
        assertEquals(5 * H, s.continuousDrivingMs)
    }

    @Test
    fun `pausa fracionada 15 + 30 repõe`() {
        val s = status(Timeline().drive(2 * H).rest(15 * M).drive(2 * H).rest(30 * M).drive(1 * H))
        assertEquals(1 * H, s.continuousDrivingMs)
    }

    @Test
    fun `pausa fracionada 30 + 15 não repõe (ordem inválida)`() {
        val s = status(Timeline().drive(2 * H).rest(30 * M).drive(1 * H).rest(15 * M).drive(1 * H))
        assertEquals(4 * H, s.continuousDrivingMs)
    }

    @Test
    fun `outro trabalho e disponibilidade não contam como pausa`() {
        val t = Timeline().drive(4 * H).work(45 * M).add(AVAILABILITY, 45 * M).drive(30 * M)
        assertEquals(4 * H + 30 * M, status(t).continuousDrivingMs)
    }

    @Test
    fun `pausa em curso mostra tempo em falta`() {
        val s = status(Timeline().drive(4 * H).rest(20 * M))
        assertEquals(45 * M, s.breakNeededMs)
        assertEquals(25 * M, s.breakRemainingMs)
        assertEquals(20 * M, s.currentRestMs)
    }

    @Test
    fun `após 1ª parte de 15 min a 2ª parte precisa de 30 min`() {
        val s = status(Timeline().drive(2 * H).rest(15 * M).drive(1 * H).rest(10 * M))
        assertEquals(30 * M, s.breakNeededMs)
        assertEquals(20 * M, s.breakRemainingMs)
    }

    // ---------- Condução diária ----------

    @Test
    fun `limite diário é 10h enquanto restam prolongamentos`() {
        val s = status(Timeline().drive(4 * H).rest(45 * M).drive(4 * H))
        assertEquals(10 * H, s.dailyLimitMs)
        assertEquals(2 * H, s.dailyRemainingMs)
    }

    @Test
    fun `após dois dias de 10h na semana o limite passa a 9h`() {
        val t = Timeline()
        repeat(2) {
            t.drive(4 * H + 30 * M).rest(45 * M).drive(4 * H + 30 * M).rest(45 * M).drive(1 * H) // 10h
            t.rest(11 * H)
        }
        t.drive(2 * H)
        val s = status(t)
        assertEquals(9 * H, s.dailyLimitMs)
        assertEquals(7 * H, s.dailyRemainingMs)
    }

    @Test
    fun `um dia de 10h usa um prolongamento e é contado na semana`() {
        val t = Timeline()
            .drive(4 * H + 30 * M).rest(45 * M).drive(4 * H + 30 * M).rest(45 * M).drive(1 * H)
            .rest(11 * H)
            .drive(1 * H)
        val s = status(t)
        assertEquals(1, s.extendedDaysThisWeek)
        assertEquals(10 * H, s.dailyLimitMs)
        assertEquals(1 * H, s.dailyDrivingMs)
    }

    @Test
    fun `descanso diário de 11h repõe o dia`() {
        val s = status(Timeline().drive(4 * H).rest(45 * M).drive(4 * H).rest(11 * H).drive(1 * H))
        assertEquals(1 * H, s.dailyDrivingMs)
        assertEquals(0, s.reducedDailyRestsUsed)
    }

    @Test
    fun `descanso reduzido de 9h repõe o dia e é contado`() {
        val s = status(Timeline().drive(4 * H).rest(9 * H).drive(1 * H))
        assertEquals(1 * H, s.dailyDrivingMs)
        assertEquals(1, s.reducedDailyRestsUsed)
    }

    @Test
    fun `descanso de 8h59 não repõe o dia`() {
        val s = status(Timeline().drive(4 * H).rest(8 * H + 59 * M).drive(1 * H))
        assertEquals(5 * H, s.dailyDrivingMs)
    }

    @Test
    fun `descanso fracionado 3h + 9h conta como regular e não como reduzido`() {
        val s = status(Timeline().drive(4 * H).rest(3 * H).drive(2 * H).rest(9 * H).drive(1 * H))
        assertEquals(1 * H, s.dailyDrivingMs)
        assertEquals(0, s.reducedDailyRestsUsed)
    }

    @Test
    fun `a condução entre as duas partes do descanso fracionado conta no mesmo dia`() {
        val s = status(Timeline().drive(4 * H).rest(3 * H).drive(2 * H))
        assertEquals(6 * H, s.dailyDrivingMs)
    }

    @Test
    fun `descanso reduzido em curso ainda não é contado`() {
        val s = status(Timeline().drive(4 * H).rest(10 * H))
        assertEquals(0, s.reducedDailyRestsUsed)
        assertEquals(0, s.dailyDrivingMs)
    }

    // ---------- Semanal e quinzenal ----------

    @Test
    fun `condução semanal acumula dentro da semana fixa`() {
        val t = Timeline()
        repeat(5) { t.drive(4 * H).rest(45 * M).drive(4 * H).rest(11 * H) } // 5 × 8h = 40h
        val s = status(t)
        assertEquals(40 * H, s.weeklyDrivingMs)
        assertEquals(16 * H, s.weeklyRemainingMs)
    }

    @Test
    fun `semana anterior conta para a quinzena mas não para a semana`() {
        // 50h na semana anterior (domingo anterior termina às 24:00), 10h nesta semana.
        val prevMonday = MONDAY - 7 * Limits.DAY
        val t = Timeline(prevMonday)
        repeat(5) { t.drive(10 * H).rest(11 * H) } // 50h dentro da semana anterior
        t.skip(MONDAY - t.end)                      // até segunda 00:00 (descanso por lacuna)
        t.drive(10 * H)
        val s = status(t)
        assertEquals(10 * H, s.weeklyDrivingMs)
        assertEquals(60 * H, s.biweeklyDrivingMs)
        assertEquals(30 * H, s.biweeklyRemainingMs)
    }

    @Test
    fun `limite quinzenal pode ser o fator limitante`() {
        val prevMonday = MONDAY - 7 * Limits.DAY
        val t = Timeline(prevMonday)
        repeat(6) { t.drive(9 * H).rest(15 * H) } // 54h na semana anterior (dias de 24h)
        t.skip(MONDAY - t.end)
        t.drive(9 * H).rest(15 * H).drive(9 * H).rest(15 * H).drive(9 * H).rest(15 * H) // 27h → 81h
        t.drive(3 * H).rest(45 * M).drive(3 * H).rest(45 * M).drive(2 * H)              // +8h → 89h
        val s = status(t)
        assertEquals(LimitKind.BIWEEKLY, s.limitingFactor)
        assertEquals(1 * H, s.biweeklyRemainingMs)
    }

    @Test
    fun `condução cortada na fronteira da semana`() {
        // Começa a conduzir no domingo às 20:00 e continua até segunda às 02:00 (6h).
        val sundayEvening = MONDAY - 4 * H
        val t = Timeline(sundayEvening).drive(6 * H)
        val s = status(t)
        assertEquals(2 * H, s.weeklyDrivingMs)    // só as 2h de segunda
        assertEquals(6 * H, s.biweeklyDrivingMs)  // 4h anteriores + 2h
    }

    // ---------- Descanso semanal e prazos ----------

    @Test
    fun `descanso de 24h conta como semanal e repõe contadores`() {
        val s = status(Timeline().drive(8 * H).rest(24 * H).drive(1 * H))
        assertFalse(s.weeklyAnchorAssumed)
        assertEquals(0, s.reducedDailyRestsUsed)
        assertEquals(1 * H, s.dailyDrivingMs)
    }

    @Test
    fun `prazo do descanso semanal é 6 períodos de 24h após o último`() {
        val t = Timeline().drive(8 * H).rest(45 * H).drive(1 * H)
        val s = status(t)
        val restEnd = MONDAY + 8 * H + 45 * H
        assertEquals(restEnd + 6 * Limits.DAY, s.weeklyRestStartByMs)
        assertFalse(s.weeklyAnchorAssumed)
    }

    @Test
    fun `sem descanso registado o prazo conta desde o primeiro registo e fica marcado`() {
        val s = status(Timeline().drive(2 * H))
        assertTrue(s.weeklyAnchorAssumed)
        assertEquals(MONDAY + 6 * Limits.DAY, s.weeklyRestStartByMs)
    }

    @Test
    fun `prazos do descanso diário`() {
        val t = Timeline().drive(8 * H).rest(11 * H).drive(1 * H)
        val s = status(t)
        val restEnd = MONDAY + 19 * H
        assertEquals(restEnd + 13 * H, s.dailyRestRegularStartByMs)
        assertEquals(restEnd + 15 * H, s.dailyRestReducedStartByMs)
    }

    @Test
    fun `alerta de descanso diário aproxima-se do prazo`() {
        val t = Timeline().drive(4 * H).rest(11 * H) // dia começa às 15:00
        t.drive(4 * H).rest(45 * M).drive(4 * H)     // até às 23:45 depois
        t.work(5 * H + 30 * M)                       // 05:15 de terça; prazo (9h reduzido) às 06:00
        val s = status(t)
        val a = s.alerts.firstOrNull { it.code == AlertCode.DAILY_REST_DEADLINE }
        assertNotNull(a)
    }

    @Test
    fun `sem alertas de condução quando em descanso`() {
        val s = status(Timeline().drive(4 * H + 29 * M).rest(5 * M))
        assertTrue(s.alerts.isEmpty())
    }

    @Test
    fun `histórico vazio devolve estado neutro`() {
        val s = engine.evaluate(emptyList(), MONDAY)
        assertNull(s.currentActivity)
        assertEquals(0, s.continuousDrivingMs)
        assertEquals(10 * H, s.dailyLimitMs)
        assertTrue(s.alerts.isEmpty())
    }

    // ---------- Linha temporal ----------

    @Test
    fun `lacunas contam como descanso`() {
        val acts = listOf(
            Activity(DRIVING, MONDAY, MONDAY + 4 * H),
            Activity(DRIVING, MONDAY + 4 * H + 45 * M, MONDAY + 5 * H + 45 * M),
        )
        val s = engine.evaluate(acts, MONDAY + 5 * H + 45 * M)
        assertEquals(1 * H, s.continuousDrivingMs)
    }

    @Test
    fun `atividade em curso conta até ao instante atual`() {
        val acts = listOf(Activity(DRIVING, MONDAY, null))
        val s = engine.evaluate(acts, MONDAY + 2 * H)
        assertEquals(2 * H, s.continuousDrivingMs)
        assertEquals(DRIVING, s.currentActivity)
    }

    @Test
    fun `a atividade mais recente prevalece em sobreposições`() {
        val acts = listOf(
            Activity(DRIVING, MONDAY, MONDAY + 3 * H),
            Activity(REST, MONDAY + 2 * H, MONDAY + 3 * H),
        )
        val s = engine.evaluate(acts, MONDAY + 3 * H)
        assertEquals(2 * H, s.dailyDrivingMs)   // a condução foi cortada às 2h
        assertEquals(0, s.continuousDrivingMs)  // e a hora de descanso repõe a contínua
        assertEquals(1 * H, s.currentRestMs)
    }

    @Test
    fun `tempo depois da última atividade fechada conta como descanso`() {
        val acts = listOf(Activity(DRIVING, MONDAY, MONDAY + 4 * H))
        val s = engine.evaluate(acts, MONDAY + 4 * H + 50 * M)
        assertEquals(REST, s.currentActivity)
        assertEquals(0, s.continuousDrivingMs)
    }

    @Test
    fun `atividades no futuro são ignoradas`() {
        val acts = listOf(Activity(DRIVING, MONDAY + 5 * H, MONDAY + 9 * H))
        val s = engine.evaluate(acts, MONDAY + 1 * H)
        assertEquals(0, s.continuousDrivingMs)
    }
}
```

## `core/src/test/kotlin/pt/tacho/core/SummariesTest.kt`

```kotlin
package pt.tacho.core

import java.time.Instant
import java.time.LocalDate
import java.time.ZoneOffset
import kotlin.test.Test
import kotlin.test.assertEquals

class SummariesTest {
    private val h = Limits.HOUR
    private val monday = Instant.parse("2026-10-19T00:00:00Z").toEpochMilli()

    @Test
    fun `atividade que passa a meia-noite é dividida pelos dois dias`() {
        val acts = listOf(Activity(ActivityType.DRIVING, monday + 22 * h, monday + 26 * h))
        val days = summarizeByDay(acts, ZoneOffset.UTC, monday + 30 * h)
        assertEquals(2, days.size)
        assertEquals(LocalDate.parse("2026-10-19"), days[0].date)
        assertEquals(2 * h, days[0].drivingMs)
        assertEquals(2 * h, days[1].drivingMs)
    }

    @Test
    fun `totais por tipo no mesmo dia`() {
        val acts = listOf(
            Activity(ActivityType.DRIVING, monday + 6 * h, monday + 10 * h),
            Activity(ActivityType.OTHER_WORK, monday + 10 * h, monday + 11 * h),
            Activity(ActivityType.AVAILABILITY, monday + 11 * h, monday + 12 * h),
            Activity(ActivityType.REST, monday + 12 * h, monday + 13 * h),
        )
        val d = summarizeByDay(acts, ZoneOffset.UTC, monday + 14 * h).single()
        assertEquals(4 * h, d.drivingMs)
        assertEquals(1 * h, d.otherWorkMs)
        assertEquals(1 * h, d.availabilityMs)
        assertEquals(1 * h, d.restMs)
    }

    @Test
    fun `condução agrupada por semana fixa`() {
        val acts = listOf(
            Activity(ActivityType.DRIVING, monday - 2 * h, monday + 3 * h), // 2h dom anterior + 3h seg
            Activity(ActivityType.DRIVING, monday + 2 * 24 * h, monday + 2 * 24 * h + 4 * h),
        )
        val weeks = summarizeDrivingByWeek(acts, ZoneOffset.UTC, monday + 5 * 24 * h)
        assertEquals(2, weeks.size)
        assertEquals(2 * h, weeks[0].second)
        assertEquals(7 * h, weeks[1].second)
    }
}
```

## `gradle.properties`

```
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
org.gradle.caching=true
android.useAndroidX=true
android.nonTransitiveRClass=true
kotlin.code.style=official
```

## `settings.gradle.kts`

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "TachoApp"

include(":core") // motor de regras (Kotlin puro, testável sem Android)
include(":app")  // aplicação Android (Jetpack Compose)
```

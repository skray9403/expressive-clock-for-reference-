# Expressive Clock source reference dump

Repository: https://github.com/skray9403/expressive-clock-for-reference-
Source branch: `main`

Complete source contents, grouped by original repository path.

## `app/build.gradle.kts`

```kotlin
import java.util.Properties

plugins {
    // No `kotlin.android`: AGP 9 has built-in Kotlin support and applying it is a hard error.
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
}

android {
    namespace = "app.expressiveclock"
    compileSdk = 37

    defaultConfig {
        applicationId = "app.expressiveclock"
        minSdk = 26
        targetSdk = 36
        versionCode = 21
        versionName = "0.1.18"
    }

    // Configure lazily enough for debug/tests/lint to work in contributor checkouts.
    // Packaging a production release is guarded explicitly; there is never a debug-key fallback.
    val keystoreProps = Properties()
    val keystorePropsFile = rootProject.file("keystore.properties")
    if (keystorePropsFile.isFile) keystorePropsFile.inputStream().use(keystoreProps::load)
    fun signingValue(name: String) = keystoreProps.getProperty(name) ?: providers.gradleProperty(name).orNull
    val releaseStore = signingValue("clockStoreFile")?.takeIf { it.isNotBlank() }?.let(rootProject::file)
    val releaseStorePassword = signingValue("clockStorePassword")
    val releaseAlias = signingValue("clockKeyAlias")
    val releaseKeyPassword = signingValue("clockKeyPassword")
    val productionSigningReady = releaseStore?.isFile == true &&
        listOf(releaseStorePassword, releaseAlias, releaseKeyPassword).all { !it.isNullOrBlank() }
    signingConfigs {
        create("release") {
            storeFile = releaseStore
            storePassword = releaseStorePassword
            keyAlias = releaseAlias
            keyPassword = releaseKeyPassword
        }
    }
    val verifyProductionSigning = tasks.register("verifyProductionSigning") {
        doLast {
            check(productionSigningReady) {
                "Production release signing requires clockStoreFile, clockStorePassword, clockKeyAlias and clockKeyPassword. No debug signing fallback is permitted."
            }
        }
    }
    tasks.matching { it.name in setOf("validateSigningRelease", "packageRelease", "signReleaseBundle", "packageReleaseBundle") }
        .configureEach { dependsOn(verifyProductionSigning) }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                // Must be the -optimize variant; plain proguard-android.txt carries -dontoptimize,
                // which is a hard error on AGP 9.
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro",
            )
        }
        debug {
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-debug"
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    buildFeatures {
        compose = true
        buildConfig = true
    }

    androidResources {
        localeFilters += listOf("en")
    }

    // No accounts, no telemetry, no ad IDs. The dependency manifest stays out of the APK too.
    dependenciesInfo {
        includeInApk = false
        includeInBundle = true
    }

    packaging {
        resources {
            excludes += setOf(
                "/META-INF/{AL2.0,LGPL2.1}",
                "/META-INF/*.version",
                "/META-INF/com.android.tools/**",
                "/kotlin/**",
                "/DebugProbesKt.bin",
            )
        }
    }

    testOptions {
        unitTests {
            isIncludeAndroidResources = true
            isReturnDefaultValues = true
        }
    }
}

kotlin {
    compilerOptions {
        // [M3E] Most expressive components are still opt-in even though MaterialExpressiveTheme
        // itself is not. Opting in centrally keeps the annotation noise out of every call site.
        optIn.addAll(
            "androidx.compose.material3.ExperimentalMaterial3Api",
            "androidx.compose.material3.ExperimentalMaterial3ExpressiveApi",
            "androidx.compose.foundation.ExperimentalFoundationApi",
            "androidx.compose.ui.text.ExperimentalTextApi",
        )
    }
}

dependencies {
    constraints {
        implementation("androidx.fragment:fragment:1.9.1") {
            because("Play in-app updates transitively requests Fragment 1.1.0, flagged as outdated by Play Console")
        }
    }
    implementation(platform(libs.androidx.compose.bom))

    implementation(libs.androidx.material3)
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.text)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.material.icons.extended)

    implementation(libs.androidx.core.ktx)
    implementation("androidx.core:core-splashscreen:1.0.1")
    implementation(libs.androidx.activity.compose)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.lifecycle.runtime.compose)

    implementation(libs.androidx.datastore.preferences)
    implementation(libs.androidx.graphics.shapes)

    implementation("com.google.android.play:app-update:2.1.0")
    implementation("com.google.android.play:app-update-ktx:2.1.0")

    debugImplementation(libs.androidx.compose.ui.tooling)

    // Real JSON codec for persistence regression tests; Android unit-test stubs are no-ops.
    testImplementation("org.json:json:20240303")
    testImplementation(libs.junit)
    testImplementation(libs.kotlin.test)
    testImplementation(libs.androidx.test.core)
}
```

---

## `app/src/main/java/app/expressiveclock/data/IslandCalibration.kt`

```kotlin
package app.expressiveclock.data

import kotlin.math.roundToInt

/** Stable preference IDs; labels and display detection never change stored keys. */
enum class IslandProfile(val label: String) {
    FOLDED_PORTRAIT("Folded Portrait"), FOLDED_LANDSCAPE("Folded Landscape"),
    UNFOLDED_PORTRAIT("Unfolded Portrait"), UNFOLDED_LANDSCAPE("Unfolded Landscape")
}

enum class IslandAlignment { LEFT, CENTRE, RIGHT }
enum class IslandExpandedPosition { BELOW_STATUS_BAR, OVERLAP_CUTOUT }

data class IslandCalibration(
    val x: Float = FACTORY_OVERLAY_OFFSET_X.toFloat(), val y: Float = FACTORY_OVERLAY_OFFSET_Y.toFloat(),
    val width: Float = FACTORY_OVERLAY_WIDTH_OFFSET.toFloat(), val height: Float = FACTORY_OVERLAY_HEIGHT_OFFSET.toFloat(),
    val enabled: Boolean = true,
    val alignment: IslandAlignment = IslandAlignment.CENTRE,
    val expandedPosition: IslandExpandedPosition = IslandExpandedPosition.OVERLAP_CUTOUT,
)

fun ThemeSettings.legacyCalibration() = IslandCalibration(overlayOffsetX.toFloat(), overlayOffsetY.toFloat(), overlayWidthOffset.toFloat(), overlayHeightOffset.toFloat())
fun ThemeSettings.calibration(profile: IslandProfile): IslandCalibration = overlayProfiles[profile] ?: legacyCalibration()
fun ThemeSettings.withCalibration(profile: IslandProfile?, value: IslandCalibration): ThemeSettings =
    if (profile == null) copy(overlayOffsetX = value.x.roundToInt(), overlayOffsetY = value.y.roundToInt(),
        overlayWidthOffset = value.width.roundToInt(), overlayHeightOffset = value.height.roundToInt())
    else copy(overlayProfiles = IslandProfile.entries.associateWith { if (it == profile) value else calibration(it) })
```

---

## `app/src/main/java/app/expressiveclock/overlay/DynamicIslandOverlay.kt`

```kotlin
package app.expressiveclock.overlay

import android.content.Intent
import android.view.WindowInsets
import android.view.WindowManager
import android.os.Build
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.DisposableEffect
import androidx.compose.ui.platform.LocalView
import androidx.compose.foundation.layout.BoxWithConstraints
import androidx.compose.foundation.layout.requiredSize
import androidx.compose.ui.draw.clipToBounds
import android.graphics.Rect
import android.util.Log
import androidx.compose.animation.AnimatedContent
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.slideInVertically
import androidx.compose.animation.slideOutVertically
import androidx.compose.animation.SizeTransform
import androidx.compose.animation.togetherWith
import androidx.compose.animation.core.Animatable
import androidx.compose.animation.core.FastOutSlowInEasing
import androidx.compose.animation.core.LinearEasing
import androidx.compose.animation.core.animateDp
import androidx.compose.animation.core.animateFloat
import androidx.compose.animation.core.spring
import androidx.compose.animation.core.tween
import androidx.compose.animation.core.updateTransition
import androidx.compose.ui.draw.blur
import androidx.compose.ui.text.font.FontFamily
import kotlinx.coroutines.delay
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.offset
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.GenericShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.outlined.CenterFocusWeak
import androidx.compose.material.icons.outlined.HourglassEmpty
import androidx.compose.material.icons.outlined.Timer
import androidx.compose.material.icons.rounded.Add
import androidx.compose.material.icons.rounded.Close
import androidx.compose.material.icons.rounded.Flag
import androidx.compose.material.icons.rounded.Pause
import androidx.compose.material.icons.rounded.PlayArrow
import androidx.compose.ui.graphics.Path
import androidx.compose.ui.graphics.Shape
import kotlin.math.cos
import kotlin.math.sin
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.geometry.CornerRadius
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.layout.onGloballyPositioned
import androidx.compose.ui.layout.positionOnScreen
import androidx.compose.ui.platform.LocalConfiguration
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.text.rememberTextMeasurer
import androidx.compose.ui.unit.Density
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.coerceIn
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.lerp
import androidx.compose.ui.unit.sp
import app.expressiveclock.MainActivity
import app.expressiveclock.alarm.Notifications
import app.expressiveclock.core.ClockTimer
import app.expressiveclock.core.Stopwatch
import app.expressiveclock.core.TimerState
import app.expressiveclock.ui.rememberElapsedTicker
import app.expressiveclock.ui.theme.ClockFace
import app.expressiveclock.ui.theme.Numerals
import java.time.Duration
import kotlin.math.roundToInt

private enum class IslandState { HIDDEN, COLLAPSED, EXPANDED }

/**
 * Dynamic Island-style overlay container and morphing UI for active Timers and Stopwatches.
 * Positioned around the status bar and camera cutout.
 *
 * Architecture reference: SmartIsland (GPL-3.0 by agupta07505).
 * Unified 3-state updateTransition machine driving geometry springs and content layers.
 */
@Composable
fun DynamicIslandOverlayContent(
    timer: ClockTimer?,
    stopwatch: Stopwatch?,
    topInsetDp: Dp,
    cutoutWidthDp: Dp = 44.dp,
    alignment: app.expressiveclock.data.IslandAlignment = app.expressiveclock.data.IslandAlignment.CENTRE,
    expandedPosition: app.expressiveclock.data.IslandExpandedPosition = app.expressiveclock.data.IslandExpandedPosition.OVERLAP_CUTOUT,
    offsetXDp: Dp = 0.dp,
    offsetYDp: Dp = 0.dp,
    widthOffsetDp: Dp = 0.dp,
    heightOffsetDp: Dp = 0.dp,
    isCalibrationPreview: Boolean = false,
    isExiting: Boolean = false,
    isExpanded: Boolean = false,
    exitRequestId: Int = 0,
    onBoundsChanged: (Rect) -> Unit = {},
    onExpandedClearanceChanged: (Float) -> Unit = {},
    onCollapsedSettled: () -> Unit = {},
    onExitComplete: () -> Unit = {},
    onExpandChanged: (Boolean) -> Unit,
    onOpenAppTab: (String) -> Unit = {},
    onPauseResumeTimer: () -> Unit,
    onAddTenTimer: () -> Unit,
    onCancelTimer: () -> Unit,
    onToggleStopwatch: () -> Unit,
    onLapStopwatch: () -> Unit,
) {
    val context = LocalContext.current

    val isTimerActive = timer != null
    val isStopwatchActive = stopwatch != null

    if (!isCalibrationPreview && !isTimerActive && !isStopwatchActive && !isExiting) return

    var isEntranceInitialized by remember { mutableStateOf(false) }
    LaunchedEffect(Unit) {
        isEntranceInitialized = true
    }

    val targetIslandState = when {
        !isEntranceInitialized -> IslandState.HIDDEN
        isExpanded && !isCalibrationPreview -> IslandState.EXPANDED
        else -> IslandState.COLLAPSED
    }

    // Dedicated Animatable for normal exit presence animation
    val exitProgress = remember { Animatable(1f) }

    LaunchedEffect(exitRequestId) {
        if (exitRequestId <= 0) return@LaunchedEffect
        Log.d("DynamicIslandOverlay", "DI EXIT EFFECT START request=$exitRequestId")
        exitProgress.snapTo(1f)
        exitProgress.animateTo(
            targetValue = 0f,
            animationSpec = spring(stiffness = 520f, dampingRatio = 0.72f),
        )
        Log.d("DynamicIslandOverlay", "DI EXIT EFFECT COMPLETE request=$exitRequestId")
        Log.d("DynamicIslandOverlay", "DI EXIT CALLING ON_EXIT_COMPLETE request=$exitRequestId")
        onExitComplete()
    }

    // Unified updateTransition state machine for normal states (HIDDEN, COLLAPSED, EXPANDED)
    val transition = updateTransition(targetState = targetIslandState, label = "SmartIslandTransition")

    // Logcat verification for entrance presence transition
    LaunchedEffect(transition.targetState) {
        if (transition.targetState == IslandState.COLLAPSED && transition.currentState == IslandState.HIDDEN) {
            Log.d("DynamicIslandOverlay", "DynamicIsland: APPEARING START")
        }
    }

    LaunchedEffect(transition.currentState, transition.targetState) {
        if (transition.currentState == IslandState.COLLAPSED) {
            Log.d("DynamicIslandOverlay", "DynamicIsland: APPEARING END")
        }
        if (transition.currentState == IslandState.COLLAPSED && transition.targetState == IslandState.COLLAPSED && !isExiting && isEntranceInitialized) {
            onCollapsedSettled()
        }
    }

    val configuration = LocalConfiguration.current
    val display by IslandDisplayMonitor.state.collectAsState()
    val screenWidthDp = (display.width / display.density).coerceAtLeast(1f).dp
    val view = LocalView.current
    var platformInsets by remember { mutableStateOf(view.rootWindowInsets) }
    DisposableEffect(view, configuration) {
        platformInsets = view.rootWindowInsets
        view.setOnApplyWindowInsetsListener { _, insets -> platformInsets = insets; insets }
        view.requestApplyInsets()
        onDispose { view.setOnApplyWindowInsetsListener(null) }
    }
    val displayDensity = LocalDensity.current.density
    val metrics = if (Build.VERSION.SDK_INT >= 30) context.getSystemService(WindowManager::class.java).currentWindowMetrics else null
    val insets = platformInsets ?: if (Build.VERSION.SDK_INT >= 30) metrics?.windowInsets else null
    val displayHeight = if (Build.VERSION.SDK_INT >= 30) metrics?.bounds?.height() ?: view.resources.displayMetrics.heightPixels else view.resources.displayMetrics.heightPixels
    val cutout = if (Build.VERSION.SDK_INT >= 28) insets?.displayCutout?.boundingRects
        ?.filter { it.top == 0 && it.width() > 0 && it.height() < displayHeight / 2 }
        ?.minByOrNull { it.top }?.let {
            IslandCutout(it.left / displayDensity, it.top / displayDensity, it.right / displayDensity, it.bottom / displayDensity)
        } else null
    val reportedStatusBar = if (Build.VERSION.SDK_INT >= 30) insets?.getInsetsIgnoringVisibility(WindowInsets.Type.statusBars())?.top
        ?.div(displayDensity) ?: 32f else topInsetDp.value.coerceAtLeast(32f)
    val statusBar = islandStatusBarHeight(reportedStatusBar, cutout).coerceAtLeast(if (cutout == null) 24f else 0f)
    val geometry = islandGeometry(screenWidthDp.value, statusBar, cutout,
        offsetXDp.value, offsetYDp.value - app.expressiveclock.data.FACTORY_OVERLAY_OFFSET_Y, widthOffsetDp.value, heightOffsetDp.value, screenHeight = display.height / display.density, alignment = alignment)
    val expandedWidth = islandExpandedWidth(screenWidthDp.value, display.height / display.density).dp
    val expandedBaseHeight = 152f + (LocalDensity.current.fontScale - 1f).coerceAtLeast(0f) * 40f
    val expandedAnchor = if (expandedPosition == app.expressiveclock.data.IslandExpandedPosition.BELOW_STATUS_BAR)
        maxOf(geometry.top, statusBar + 6f) else geometry.top
    val expandedTop = islandExpandedTop(expandedAnchor, display.height / display.density, expandedBaseHeight).dp
    val expandedClearance = expandedContentClearance(expandedPosition, statusBar, cutout?.bottom ?: 0f, expandedTop.value)
    LaunchedEffect(expandedClearance) { onExpandedClearanceChanged(expandedClearance) }
    val expandedHeight = (expandedClearance + expandedBaseHeight).dp
    val collapsedWidth = geometry.width.dp
    val collapsedHeight = geometry.height.dp
    val islandTop by transition.animateDp(label = "islandTop", transitionSpec = { spring(stiffness = 520f, dampingRatio = .72f) }) {
        if (it == IslandState.EXPANDED) expandedTop else geometry.top.dp
    }
    val islandOffsetX by transition.animateDp(label = "islandCenter", transitionSpec = { spring(stiffness = 520f, dampingRatio = .72f) }) {
        if (it == IslandState.EXPANDED) 0.dp else (geometry.centerX - screenWidthDp.value / 2f).dp
    }

    // Synchronized geometry spring spec (stiffness = 520f, dampingRatio = 0.72f)
    val geometrySpringDp = spring<Dp>(
        stiffness = 520f,
        dampingRatio = 0.72f,
    )

    val cameraCutoutDiameter = cutoutWidthDp.coerceAtLeast(36.dp)

    // Master presence progress for shape convergence guard
    val transitionPresenceProgress by transition.animateFloat(
        label = "presenceProgress",
        transitionSpec = { spring(stiffness = 520f, dampingRatio = 0.72f) },
    ) { state ->
        when (state) {
            IslandState.HIDDEN -> 0f
            IslandState.COLLAPSED -> 1f
            IslandState.EXPANDED -> 1f
        }
    }

    val presenceProgress = if (isExiting) exitProgress.value else transitionPresenceProgress

    val visualWidth by transition.animateDp(
        label = "visualWidth",
        transitionSpec = { geometrySpringDp },
    ) { state ->
        when (state) {
            IslandState.HIDDEN -> 1.dp
            IslandState.COLLAPSED -> collapsedWidth
            IslandState.EXPANDED -> expandedWidth
        }
    }

    val visualHeight by transition.animateDp(
        label = "visualHeight",
        transitionSpec = { geometrySpringDp },
    ) { state ->
        when (state) {
            IslandState.HIDDEN -> 1.dp
            IslandState.COLLAPSED -> collapsedHeight
            IslandState.EXPANDED -> expandedHeight
        }
    }

    val currentCornerRadius by transition.animateDp(
        label = "cornerRadius",
        transitionSpec = { geometrySpringDp },
    ) { state ->
        when (state) {
            IslandState.HIDDEN -> 0.5.dp
            IslandState.COLLAPSED -> (collapsedHeight / 2f)
            IslandState.EXPANDED -> 42.dp
        }
    }

    // Direct geometry mapping: Pill contracts smoothly to 1dp circle
    val clampedProgress = presenceProgress.coerceIn(0f, 1f)
    val effectiveWidth: Dp
    val effectiveHeight: Dp
    val effectiveCornerRadius: Dp

    if (!isExiting && (transition.currentState == IslandState.EXPANDED || transition.targetState == IslandState.EXPANDED)) {
        // Collapsed <-> Expanded morph
        effectiveWidth = visualWidth
        effectiveHeight = visualHeight
        effectiveCornerRadius = currentCornerRadius
    } else {
        // Proportional contraction towards 1dp circle
        val endpointDiameter = 1.dp
        effectiveWidth = lerp(endpointDiameter, collapsedWidth, clampedProgress)
        effectiveHeight = lerp(endpointDiameter, collapsedHeight, clampedProgress)
        effectiveCornerRadius = lerp(0.5.dp, collapsedHeight / 2f, clampedProgress)
    }

    // Optional final pill surface fade over the final ~6dp endpoint
    val currentAlpha = (effectiveWidth / 6.dp).coerceIn(0f, 1f)

    // Inner content fast fade for pill contraction towards 1dp (~133ms / 0.35 progress fraction)
    val fastContentAlpha = ((clampedProgress - 0.65f) / 0.35f).coerceIn(0f, 1f)

    // Layer 2: Collapsed Content Fade & Blur (8 frames / 133ms duration)
    val transitionCollapsedAlpha by transition.animateFloat(
        label = "collapsedAlpha",
        transitionSpec = {
            if (targetState == IslandState.COLLAPSED) {
                if (initialState == IslandState.EXPANDED) {
                    tween(durationMillis = 133, delayMillis = 167, easing = FastOutSlowInEasing)
                } else {
                    tween(durationMillis = 133, delayMillis = 150, easing = FastOutSlowInEasing)
                }
            } else if (targetState == IslandState.EXPANDED) {
                tween(durationMillis = 133, easing = FastOutSlowInEasing)
            } else {
                tween(durationMillis = 133, easing = FastOutSlowInEasing)
            }
        },
    ) { state ->
        if (state == IslandState.COLLAPSED) 1f else 0f
    }

    val collapsedBlur by transition.animateDp(
        label = "collapsedBlur",
        transitionSpec = {
            if (targetState == IslandState.COLLAPSED) {
                if (initialState == IslandState.EXPANDED) {
                    tween(durationMillis = 133, delayMillis = 167, easing = FastOutSlowInEasing)
                } else {
                    tween(durationMillis = 133, delayMillis = 150, easing = FastOutSlowInEasing)
                }
            } else {
                tween(durationMillis = 133, easing = FastOutSlowInEasing)
            }
        },
    ) { state ->
        if (state == IslandState.COLLAPSED) 0.dp else 6.dp
    }

    val collapsedAlpha = if (isExiting) {
        fastContentAlpha
    } else {
        transitionCollapsedAlpha * fastContentAlpha
    }

    // Layer 3: Incoming Expanded Content Fade & Blur (8 frames / 133ms duration)
    val expandedAlpha by transition.animateFloat(
        label = "expandedAlpha",
        transitionSpec = {
            if (targetState == IslandState.EXPANDED) {
                tween(durationMillis = 133, delayMillis = 167, easing = FastOutSlowInEasing)
            } else {
                tween(durationMillis = 133, easing = FastOutSlowInEasing)
            }
        },
    ) { state ->
        if (state == IslandState.EXPANDED) 1f else 0f
    }

    val expandedBlur by transition.animateDp(
        label = "expandedBlur",
        transitionSpec = {
            if (targetState == IslandState.EXPANDED) {
                tween(durationMillis = 133, delayMillis = 167, easing = FastOutSlowInEasing)
            } else {
                tween(durationMillis = 133, easing = FastOutSlowInEasing)
            }
        },
    ) { state ->
        if (state == IslandState.EXPANDED) 0.dp else 6.dp
    }

    fun openAppTab(tab: String) {
        onOpenAppTab(tab)
    }

    val density = LocalDensity.current
    val topInsetPx = with(density) { islandTop.toPx() }
    val offsetXPx = with(density) { islandOffsetX.toPx() }
    val offsetYPx = 0f
    val collapsedHeightPx = with(density) { collapsedHeight.toPx() }
    val effectiveWidthPx = with(density) { effectiveWidth.toPx() }
    val effectiveHeightPx = with(density) { effectiveHeight.toPx() }
    val effectiveCornerRadiusPx = with(density) { effectiveCornerRadius.toPx() }

    // Outer root container
    Box(
        modifier = Modifier
            .fillMaxSize()
            .then(
                if (isExpanded && !isCalibrationPreview) {
                    Modifier.pointerInput(Unit) {
                        detectTapGestures {
                            // Tap outside expanded card collapses back to capsule
                            onExpandChanged(false)
                        }
                    }
                } else Modifier
            ),
        contentAlignment = Alignment.TopCenter,
    ) {
        // Layer A: Full-screen Canvas drawing the single black surface
        Canvas(
            modifier = Modifier
                .fillMaxSize()
                .graphicsLayer {
                    this.alpha = currentAlpha
                },
        ) {
            // Exact positioning: top-aligned for Collapsed <-> Expanded morph, center-anchored for contraction
            val screenCenterX = size.width / 2f
            val islandLeftPx = screenCenterX + offsetXPx - (effectiveWidthPx / 2f)
            val islandTopPx = if (!isExiting && (transition.currentState == IslandState.EXPANDED || transition.targetState == IslandState.EXPANDED)) {
                topInsetPx + offsetYPx
            } else {
                topInsetPx + offsetYPx + ((collapsedHeightPx - effectiveHeightPx) / 2f)
            }

            drawRoundRect(
                color = Color.Black,
                topLeft = Offset(islandLeftPx, islandTopPx),
                size = Size(effectiveWidthPx, effectiveHeightPx),
                cornerRadius = CornerRadius(effectiveCornerRadiusPx, effectiveCornerRadiusPx),
            )
        }

        // Layer B: Content Box positioned over the drawn black island surface
        Box(
            modifier = Modifier
                // Position is translation, not padding: a spring's brief negative excursion
                // during a display/profile transition must not create illegal padding.
                .offset { androidx.compose.ui.unit.IntOffset(islandOffsetX.roundToPx(), islandTop.roundToPx()) }
                .width(effectiveWidth)
                .height(effectiveHeight)
                .clipToBounds()
                .onGloballyPositioned { coordinates ->
                    val pos = coordinates.positionOnScreen()
                    val size = coordinates.size
                    onBoundsChanged(
                        Rect(
                            pos.x.roundToInt(),
                            pos.y.roundToInt(),
                            (pos.x + size.width).roundToInt(),
                            (pos.y + size.height).roundToInt(),
                        )
                    )
                }
                .clickable(
                    enabled = !isCalibrationPreview,
                    indication = null,
                    interactionSource = remember { MutableInteractionSource() },
                ) {
                    if (isExpanded) {
                        // Tapping the main black island surface opens the Clock app to the active tab
                        openAppTab(if (isTimerActive) Notifications.TAB_TIMERS else Notifications.TAB_STOPWATCH)
                    } else if (!isExiting) {
                        onExpandChanged(true)
                    }
                },
            contentAlignment = Alignment.Center,
        ) {
            if (isCalibrationPreview) {
                CalibrationTargetCapsule(cutoutWidthDp = cutoutWidthDp)
            } else {
                Box(
                    modifier = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center,
                ) {
                    // Layer 2: Collapsed Content (Fixed unpadded bounds: never squished or clipped during expansion)
                    if (collapsedAlpha > 0.01f) {
                        Box(
                            modifier = Modifier
                                .width(collapsedWidth)
                                .height(collapsedHeight)
                                .blur(collapsedBlur)
                                .graphicsLayer {
                                    alpha = collapsedAlpha
                                },
                            contentAlignment = Alignment.Center,
                        ) {
                            CollapsedIslandCapsule(
                                timer = timer,
                                stopwatch = stopwatch,
                                cutoutWidthDp = cutoutWidthDp,
                            )
                        }
                    }

                    // Layer 3: Expanded Content (Fixed expanded bounds: controls never shift horizontally during contraction)
                    if (expandedAlpha > 0.01f) {
                        Box(
                            modifier = Modifier
                                .requiredSize(expandedWidth, expandedHeight)
                                .blur(expandedBlur)
                                .graphicsLayer {
                                    alpha = expandedAlpha
                                }
                                .padding(horizontal = (expandedWidth.value * .055f).coerceIn(10f, 24f).dp)
                                .padding(top = 16.dp + expandedClearance.dp, bottom = 16.dp),
                            contentAlignment = Alignment.Center,
                        ) {
                            ExpandedIslandCard(
                                timer = timer,
                                stopwatch = stopwatch,
                                onOpenTimerTab = { openAppTab(Notifications.TAB_TIMERS) },
                                onOpenStopwatchTab = { openAppTab(Notifications.TAB_STOPWATCH) },
                                onPauseResumeTimer = onPauseResumeTimer,
                                onAddTenTimer = onAddTenTimer,
                                onCancelTimer = onCancelTimer,
                                onToggleStopwatch = onToggleStopwatch,
                                onLapStopwatch = onLapStopwatch,
                            )
                        }
                    }
                }
            }
        }
    }
}

@Composable
private fun CalibrationTargetCapsule(
    cutoutWidthDp: Dp,
) {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        // Outer faded ring/circle (matches MT Capsule calibration visual reference)
        Box(
            modifier = Modifier
                .size(32.dp)
                .clip(CircleShape)
                .background(Color.White.copy(alpha = 0.22f))
                .border(1.dp, Color.White.copy(alpha = 0.35f), CircleShape),
            contentAlignment = Alignment.Center,
        ) {
            // Bright red inner target circle with white border stroke
            Box(
                modifier = Modifier
                    .size(20.dp)
                    .clip(CircleShape)
                    .background(Color(0xFFE53935))
                    .border(1.5.dp, Color.White, CircleShape),
            )
        }
    }
}

@Composable
private fun CompactTickerText(
    valueString: String,
    fontFamily: FontFamily,
    modifier: Modifier = Modifier,
    countDown: Boolean = true,
) {
    val roll = mechanicalRollEnabled(valueString)
    val down = mechanicalCountDown(valueString, countDown)
    Row(modifier, verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.End) {
        valueString.forEachIndexed { index, char ->
            val cell: @Composable (Char) -> Unit = { digit ->
                Text(digit.toString(), style = MaterialTheme.typography.labelLarge.copy(
                    fontWeight = FontWeight.Bold, fontSize = 13.sp, fontFamily = fontFamily), color = Color.White)
            }
            if (char.isDigit()) MechanicalDigit(char, index, valueString.length, down, roll, content = cell)
            else cell(char)
        }
    }
}

@Composable
private fun CollapsedIslandCapsule(
    timer: ClockTimer?,
    stopwatch: Stopwatch?,
    cutoutWidthDp: Dp,
) {
    val isTimer = timer != null
    val nowElapsed by rememberElapsedTicker(active = isTimer || (stopwatch != null && stopwatch.running))
    val compactFontFamily = remember { ClockFace.family(opticalSize = 14f, width = 65f, weight = 700) }

    val iconRotationZ = remember { Animatable(0f) }
    LaunchedEffect(Unit) {
        while (true) {
            delay(5000L)
            iconRotationZ.animateTo(
                targetValue = 360f,
                animationSpec = tween(durationMillis = 450, easing = FastOutSlowInEasing),
            )
            iconRotationZ.snapTo(0f)
        }
    }

    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        if (isTimer && timer != null) {
            val remaining = timer.remaining(nowElapsed)
            // Left side: Icon anchored 6dp from left inner edge
            Icon(
                Icons.Outlined.HourglassEmpty,
                contentDescription = null,
                tint = Color.White,
                modifier = Modifier
                    .align(Alignment.CenterStart)
                    .padding(start = 6.dp)
                    .size(16.dp)
                    .graphicsLayer {
                        rotationZ = iconRotationZ.value
                    },
            )

            // Right side: Compact ticker text anchored 6dp from right inner edge
            CompactTickerText(
                valueString = formatCompactDuration(remaining),
                fontFamily = compactFontFamily,
                modifier = Modifier
                    .align(Alignment.CenterEnd)
                    .padding(end = 6.dp),
            )
        } else if (stopwatch != null) {
            val elapsed = stopwatch.elapsed(nowElapsed)
            // Left side: Icon anchored 6dp from left inner edge
            Icon(
                Icons.Outlined.Timer,
                contentDescription = null,
                tint = Color.White,
                modifier = Modifier
                    .align(Alignment.CenterStart)
                    .padding(start = 6.dp)
                    .size(16.dp)
                    .graphicsLayer {
                        rotationZ = iconRotationZ.value
                    },
            )

            // Right side: Compact ticker text anchored 6dp from right inner edge
            CompactTickerText(
                valueString = formatCompactDuration(elapsed),
                countDown = false,
                fontFamily = compactFontFamily,
                modifier = Modifier
                    .align(Alignment.CenterEnd)
                    .padding(end = 6.dp),
            )
        }
    }
}

/**
 * Material 3 Expressive circular scalloped shape with 12 subtle rounded lobes.
 */
private val ScallopedShape: Shape = GenericShape { size, _ ->
    val centerX = size.width / 2f
    val centerY = size.height / 2f
    val outerRadius = minOf(size.width, size.height) / 2f
    val innerRadius = outerRadius * 0.91f
    val lobes = 12
    val angleStep = (2 * Math.PI / lobes).toFloat()

    for (i in 0 until lobes) {
        val angle = i * angleStep
        val nextAngle = (i + 1) * angleStep
        val midAngle = angle + angleStep / 2f

        val xOuter = centerX + outerRadius * cos(midAngle)
        val yOuter = centerY + outerRadius * sin(midAngle)
        val xInner = centerX + innerRadius * cos(nextAngle)
        val yInner = centerY + innerRadius * sin(nextAngle)

        if (i == 0) {
            val xStart = centerX + innerRadius * cos(0f)
            val yStart = centerY + innerRadius * sin(0f)
            moveTo(xStart, yStart)
        }

        quadraticTo(xOuter, yOuter, xInner, yInner)
    }
    close()
}

@Composable
private fun ExpressiveButton(
    containerColor: Color,
    onClick: () -> Unit,
    buttonSize: Dp = 62.dp,
    content: @Composable () -> Unit,
) {
    Box(
        modifier = Modifier
            .size(buttonSize)
            .clip(ScallopedShape)
            .background(containerColor)
            .clickable(onClick = onClick),
        contentAlignment = Alignment.Center,
    ) {
        content()
    }
}

@Composable
internal fun ExpandedIslandCard(
    timer: ClockTimer?,
    stopwatch: Stopwatch?,
    onOpenTimerTab: () -> Unit,
    onOpenStopwatchTab: () -> Unit,
    onPauseResumeTimer: () -> Unit,
    onAddTenTimer: () -> Unit,
    onCancelTimer: () -> Unit,
    onToggleStopwatch: () -> Unit,
    onLapStopwatch: () -> Unit,
) {
    val isTimer = timer != null && timer.state != TimerState.FINISHED
    val nowElapsed by rememberElapsedTicker(active = isTimer || (stopwatch != null && stopwatch.running))

    val value = formatFullDuration(if (isTimer && timer != null) timer.remaining(nowElapsed) else stopwatch?.elapsed(nowElapsed) ?: Duration.ZERO)
    val density = LocalDensity.current
    val measurer = rememberTextMeasurer()
    BoxWithConstraints(Modifier.fillMaxSize()
        .clickable(onClick = if (isTimer) onOpenTimerTab else onOpenStopwatchTab)) {
        // Measure each cell, including the widest digit, with the same physical
        // cap-height typography used by the ticker. Reserve inter-cell spacing too.
        val neutralDensity = Density(density.density, 1f)
        val sizing = remember(maxWidth, value.length, density.density, measurer) {
            fun widthAt(cap: Float): Float {
                val style = ClockFace.numerals(cap.dp, ClockFace.CONDENSED, 500, true, ClockFace.CONDENSED_TRACKING, false)
                val digit = ('0'..'9').maxOf { measurer.measure(it.toString(), style, density = neutralDensity).size.width }
                val colon = measurer.measure(":", style, density = neutralDensity).size.width
                return (value.count { it.isDigit() } * digit + value.count { !it.isDigit() } * colon) / density.density + (value.length - 1) * 1.5f
            }
            val layout = expandedCardLayout(maxWidth.value, widthAt(38f), widthAt(20f))
            var low = 1f
            var high = 38f
            repeat(12) {
                val middle = (low + high) / 2f
                if (widthAt(middle) <= layout.readoutWidth - 2f) low = middle else high = middle
            }
            layout to low
        }
        val layout = sizing.first
        val capHeight = sizing.second
        val readout: @Composable () -> Unit = {
            Column(Modifier.width(layout.readoutWidth.dp), verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.Start) {
                // Numerals use a physical cap height; keep font scaling from breaking
                // that ink box. Captions and control labels still respect accessibility scale.
                CompositionLocalProvider(LocalDensity provides neutralDensity) {
                    TickerTimerReadout(text = value, countDown = isTimer, capHeight = capHeight.dp, color = Color.White)
                }
                Spacer(Modifier.height(2.dp))
                androidx.compose.foundation.text.BasicText(
                    if (isTimer) "Time remaining" else "Elapsed time",
                    style = MaterialTheme.typography.labelMedium.copy(fontWeight = FontWeight.Medium,
                        fontSize = 13.sp, color = Color.White.copy(alpha = .65f)),
                    maxLines = 2,
                    autoSize = androidx.compose.foundation.text.TextAutoSize.StepBased(
                        minFontSize = 10.sp, maxFontSize = 13.sp, stepSize = .25.sp))
            }
        }
        val controls: @Composable () -> Unit = {
            Row(
                modifier = Modifier,
                horizontalArrangement = Arrangement.spacedBy(layout.controlGap.dp),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                if (isTimer && timer != null) {
                    val running = timer.state == TimerState.RUNNING
                    // Button 1: Pause / Resume
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFF2C3242),
                        onClick = onPauseResumeTimer,
                    ) {
                        Icon(
                            imageVector = if (running) Icons.Rounded.Pause else Icons.Rounded.PlayArrow,
                            contentDescription = if (running) "Pause" else "Resume",
                            tint = Color.White,
                            modifier = Modifier.size(28.dp),
                        )
                    }
                    // Button 2: +1m
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFFD6E2FF),
                        onClick = onAddTenTimer,
                    ) {
                        androidx.compose.foundation.text.BasicText(
                            text = "+1m",
                            style = MaterialTheme.typography.titleMedium.copy(
                                fontWeight = FontWeight.Bold, fontSize = 17.sp, color = Color(0xFF0F1A2D)),
                            modifier = Modifier.padding(horizontal = 8.dp),
                            maxLines = 1, softWrap = false,
                            autoSize = androidx.compose.foundation.text.TextAutoSize.StepBased(
                                minFontSize = 8.sp, maxFontSize = 17.sp, stepSize = .25.sp),
                        )
                    }
                    // Button 3: Cancel
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFF5B2B31),
                        onClick = onCancelTimer,
                    ) {
                        Icon(
                            imageVector = Icons.Rounded.Close,
                            contentDescription = "Cancel",
                            tint = Color(0xFFFFB4AB),
                            modifier = Modifier.size(28.dp),
                        )
                    }
                } else if (stopwatch != null) {
                    val running = stopwatch.running
                    // Button 1: Pause / Resume
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFF2C3242),
                        onClick = onToggleStopwatch,
                    ) {
                        Icon(
                            imageVector = if (running) Icons.Rounded.Pause else Icons.Rounded.PlayArrow,
                            contentDescription = if (running) "Pause" else "Start",
                            tint = Color.White,
                            modifier = Modifier.size(28.dp),
                        )
                    }
                    // Button 2: Lap
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFFD6E2FF),
                        onClick = onLapStopwatch,
                    ) {
                        Icon(
                            imageVector = Icons.Rounded.Flag,
                            contentDescription = "Lap",
                            tint = Color(0xFF0F1A2D),
                            modifier = Modifier.size(28.dp),
                        )
                    }
                    // Button 3: Cancel
                    ExpressiveButton(
                        buttonSize = layout.button.dp,
                        containerColor = Color(0xFF5B2B31),
                        onClick = onCancelTimer,
                    ) {
                        Icon(
                            imageVector = Icons.Rounded.Close,
                            contentDescription = "Cancel",
                            tint = Color(0xFFFFB4AB),
                            modifier = Modifier.size(28.dp),
                        )
                    }
                }
            }
        }
        if (layout.stacked) {
            // Only exceptionally narrow cards use a compact version of the same
            // hierarchy: left readout above right-aligned controls.
            Column(Modifier.fillMaxSize(), verticalArrangement = Arrangement.spacedBy(8.dp, Alignment.CenterVertically)) {
                readout()
                Box(Modifier.fillMaxWidth(), contentAlignment = Alignment.CenterEnd) { controls() }
            }
        } else {
            Row(Modifier.fillMaxSize(), horizontalArrangement = Arrangement.spacedBy(layout.groupGap.dp),
                verticalAlignment = Alignment.CenterVertically) {
                readout()
                controls()
            }
        }
    }
}

/**
 * Ticker readout that always formats as 00:00:00 (6 digits) and uses digit-by-digit overshoot animations.
 */
@Composable
private fun TickerTimerReadout(
    text: String,
    capHeight: Dp = 38.dp,
    color: Color = Color.White,
    countDown: Boolean = true,
) {
    val roll = mechanicalRollEnabled(text)
    val down = mechanicalCountDown(text, countDown)
    Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(1.5.dp)) {
        text.forEachIndexed { index, char ->
            val cell: @Composable (Char) -> Unit = { digit ->
                Numerals(text = digit.toString(), capHeight = capHeight, color = color,
                    width = ClockFace.CONDENSED, weight = 500, tracking = ClockFace.CONDENSED_TRACKING)
            }
            if (char.isDigit()) MechanicalDigit(char, index, text.length, down, roll, content = cell)
            else cell(char)
        }
    }
}

internal fun formatCompactDuration(duration: Duration): String {
    val seconds = duration.seconds.coerceAtLeast(0)
    return when {
        seconds >= 3600 -> "%02dh".format(seconds / 3600)
        seconds >= 60 -> "%02dm".format(seconds / 60)
        else -> "%02ds".format(seconds)
    }
}

private fun formatFullDuration(duration: Duration): String {
    val totalSecs = duration.seconds.coerceAtLeast(0)
    val hrs = totalSecs / 3600
    val mins = (totalSecs % 3600) / 60
    val secs = totalSecs % 60
    return "%02d:%02d:%02d".format(hrs, mins, secs)
}


@Composable
internal fun calibrationVerticalRange(heightAdjustment: Float): ClosedFloatingPointRange<Float> {
    val display by IslandDisplayMonitor.state.collectAsState()
    val view = LocalView.current
    val context = LocalContext.current
    val density = LocalDensity.current.density
    val configuration = LocalConfiguration.current
    var platformInsets by remember { mutableStateOf(view.rootWindowInsets) }
    DisposableEffect(view, configuration) {
        platformInsets = view.rootWindowInsets
        view.setOnApplyWindowInsetsListener { _, insets -> platformInsets = insets; insets }
        view.requestApplyInsets()
        onDispose { view.setOnApplyWindowInsetsListener(null) }
    }
    val metrics = if (Build.VERSION.SDK_INT >= 30) context.getSystemService(WindowManager::class.java).currentWindowMetrics else null
    val insets = platformInsets ?: if (Build.VERSION.SDK_INT >= 30) metrics?.windowInsets else null
    val cutout = if (Build.VERSION.SDK_INT >= 28) insets?.displayCutout?.boundingRects
        ?.firstOrNull { it.top == 0 && it.width() > 0 && it.height() < display.height / 2 }
        ?.let { IslandCutout(it.left / density, it.top / density, it.right / density, it.bottom / density) } else null
    val reported = if (Build.VERSION.SDK_INT >= 30)
        insets?.getInsetsIgnoringVisibility(WindowInsets.Type.statusBars())?.top?.div(density) ?: 32f else 32f
    val statusBar = islandStatusBarHeight(reported, cutout).coerceAtLeast(if (cutout == null) 24f else 0f)
    return islandVerticalRange(display.width / display.density, display.height / display.density,
        statusBar, cutout, heightAdjustment)
}
```

---

## `app/src/main/java/app/expressiveclock/overlay/FullscreenVisibility.kt`

```kotlin
package app.expressiveclock.overlay

/** Unknown visibility is deliberately not evidence of immersive fullscreen. */
internal fun shouldHideIslandForFullscreen(statusBarVisible: Boolean?, navigationBarVisible: Boolean?,
    locked: Boolean, calibrationPreview: Boolean): Boolean =
    !locked && !calibrationPreview && statusBarVisible == false && navigationBarVisible == false
```


---

## `app/src/main/java/app/expressiveclock/overlay/IslandDisplayMonitor.kt`

```kotlin
package app.expressiveclock.overlay

import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.hardware.display.DisplayManager
import android.os.Handler
import android.os.Looper
import android.util.DisplayMetrics
import android.view.Display
import app.expressiveclock.data.IslandProfile
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow

internal fun islandProfile(foldable: Boolean, unfolded: Boolean, width: Int, height: Int): IslandProfile? {
    if (!foldable) return null
    return when {
        unfolded && width > height -> IslandProfile.UNFOLDED_LANDSCAPE
        unfolded -> IslandProfile.UNFOLDED_PORTRAIT
        width > height -> IslandProfile.FOLDED_LANDSCAPE
        else -> IslandProfile.FOLDED_PORTRAIT
    }
}

data class IslandDisplayState(val width: Int = 0, val height: Int = 0, val density: Float = 1f,
    val rotation: Int = 0, val displayId: Int = 0, val foldable: Boolean = false, val unfolded: Boolean = false) {
    val calibrationProfile get() = islandProfile(true, unfolded, width, height)!!
    val profile get() = islandProfile(foldable, unfolded, width, height)
}

/** Process lifetime observer, shared by Activity settings and the independently rotating service window. */
object IslandDisplayMonitor {
    private val mutable = MutableStateFlow(IslandDisplayState())
    val state = mutable.asStateFlow()
    private var context: Context? = null
    private var hingeAngle: Float? = null
    private var hasHingeSensor = false
    private val main = Handler(Looper.getMainLooper())
    fun start(ctx: Context) {
        if (context != null) return
        context = ctx.applicationContext
        ctx.getSystemService(DisplayManager::class.java).registerDisplayListener(object : DisplayManager.DisplayListener {
            override fun onDisplayAdded(id: Int) = refresh()
            override fun onDisplayRemoved(id: Int) = refresh()
            override fun onDisplayChanged(id: Int) = refresh()
        }, main)
        val sensors = ctx.getSystemService(SensorManager::class.java)
        // TYPE_HINGE_ANGLE is 36 (API30); numeric ID keeps the API26 fallback available.
        val hinge = sensors.getDefaultSensor(36)
        hasHingeSensor = hinge != null
        hinge?.let { sensor ->
            sensors.registerListener(object : SensorEventListener {
                override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) = Unit
                override fun onSensorChanged(event: SensorEvent) { hingeAngle = event.values.firstOrNull(); refresh() }
            }, sensor, SensorManager.SENSOR_DELAY_NORMAL, main)
        }
        refresh()
    }
    @Suppress("DEPRECATION")
    fun refresh() {
        val ctx = context ?: return
        val display = ctx.getSystemService(DisplayManager::class.java).getDisplay(Display.DEFAULT_DISPLAY) ?: return
        val metrics = DisplayMetrics(); display.getRealMetrics(metrics)
        val foldable = hasHingeSensor || ctx.packageManager.hasSystemFeature("android.hardware.sensor.hinge_angle") ||
            ctx.packageManager.hasSystemFeature("android.hardware.type.foldable")
        // Hinge posture is primary. Only known foldables use the window-size fallback while it is unavailable.
        val unfolded = foldable && (hingeAngle?.let { it >= 30f } ?: (minOf(metrics.widthPixels, metrics.heightPixels) / metrics.density >= 600f))
        mutable.value = IslandDisplayState(metrics.widthPixels, metrics.heightPixels, metrics.density,
            display.rotation, display.displayId, foldable, unfolded)
    }
}
```


---

## `app/src/main/java/app/expressiveclock/overlay/IslandGeometry.kt`

```kotlin
package app.expressiveclock.overlay

/** Coordinates are in density-independent display space; system icon positions are not public API. */
internal data class IslandCutout(val left: Float, val top: Float, val right: Float, val bottom: Float)
internal data class IslandGeometry(val centerX: Float, val top: Float, val width: Float, val height: Float, val belowStatusBar: Boolean)

internal fun islandGeometry(
    screenWidth: Float, statusBar: Float, cutout: IslandCutout?,
    offsetX: Float = 0f, offsetY: Float = 0f, widthAdjustment: Float = 0f, heightAdjustment: Float = 0f,
    screenHeight: Float = Float.POSITIVE_INFINITY,
    alignment: app.expressiveclock.data.IslandAlignment = app.expressiveclock.data.IslandAlignment.CENTRE,
): IslandGeometry {
    val width = screenWidth.coerceAtLeast(1f)
    val cameraWidth = cutout?.let { it.right - it.left } ?: 0f
    val height = (38f + heightAdjustment).coerceIn(32f, 60f)
    val desired = (96f + cameraWidth + widthAdjustment).coerceAtLeast(108f).coerceAtMost((width - 16f).coerceAtLeast(1f))
    // Reserve the outer status-bar regions for clock, notifications and system indicators.
    // These are conservative exclusion zones, not a claim to know Android's icon bounds.
    val safeLeft = width * .36f
    val safeRight = width * .64f
    val cameraX = cutout?.let { (it.left + it.right) / 2f } ?: width / 2f
    val cameraY = cutout?.let { (it.top + it.bottom) / 2f } ?: -1f
    val available = 2f * minOf(cameraX - safeLeft, safeRight - cameraX)
    val minimum = maxOf(96f, cameraWidth + 48f)
    // A top-edge cutout rectangle includes the space above the physical lens.
    // Requiring the capsule to fit inside that rectangle/status bar incorrectly
    // sends shallow cutouts below the camera. Keep horizontal icon clearance,
    // and clamp the camera-centred capsule against the actual display edge.
    val fits = cutout != null && available >= minimum
    // Automatic sizing reserves icon space. Explicit enlargement honours calibration,
    // while the display-edge clamp still keeps the complete capsule reachable.
    val pillWidth = if (fits && widthAdjustment <= app.expressiveclock.data.FACTORY_OVERLAY_WIDTH_OFFSET)
        minOf(desired, available) else desired
    val anchor = when (alignment) {
        app.expressiveclock.data.IslandAlignment.LEFT -> width * .18f
        app.expressiveclock.data.IslandAlignment.RIGHT -> width * .82f
        app.expressiveclock.data.IslandAlignment.CENTRE -> if (fits) cameraX else width / 2f
    }
    val center = (anchor + offsetX)
        .coerceIn(pillWidth / 2f, width - pillWidth / 2f)
    val top = if (fits) cameraY - height / 2f else maxOf(statusBar, cutout?.bottom ?: 0f) + 6f
    val minTop = 0f
    val maxTop = (screenHeight - height - 8f).coerceAtLeast(minTop)
    return IslandGeometry(center, (top.coerceAtLeast(0f) + offsetY).coerceIn(minTop, maxTop), pillWidth, height, !fits)
}

internal fun islandReadoutCap(width: Float, characters: Int): Float =
    minOf(38f, width.coerceAtLeast(1f) / characters.coerceAtLeast(1)).coerceAtLeast(1f)

/** Full-screen accessibility overlays may receive zero status-bar insets. */
internal fun islandStatusBarHeight(reported: Float, cutout: IslandCutout?): Float =
    maxOf(reported, cutout?.bottom ?: 0f)

internal fun islandExpandedTop(top: Float, screenHeight: Float, contentHeight: Float): Float =
    top.coerceIn(8f, (screenHeight - contentHeight - 8f).coerceAtLeast(8f))

/** A portrait-style control card, never a full-width landscape banner. */
internal fun islandExpandedWidth(width: Float, height: Float): Float =
    minOf(width - 24f, height - 24f, 420f).coerceAtLeast(1f)

/** Offset range corresponding exactly to the capsule's reachable display positions. */
internal fun islandVerticalRange(width: Float, height: Float, statusBar: Float,
    cutout: IslandCutout?, heightAdjustment: Float): ClosedFloatingPointRange<Float> {
    val base = islandGeometry(width, statusBar, cutout, heightAdjustment = heightAdjustment, screenHeight = height)
    return -base.top..((height - base.height - 8f).coerceAtLeast(0f) - base.top)
}

/** Centre is the default; each side uses only its physically available travel. */
internal fun verticalSliderPosition(offset: Float, minimum: Float, maximum: Float): Float =
    if (offset < 0f && minimum < 0f) (-offset / minimum).coerceIn(-1f, 0f)
    else if (offset > 0f && maximum > 0f) (offset / maximum).coerceIn(0f, 1f) else 0f

internal fun verticalSliderOffset(position: Float, minimum: Float, maximum: Float): Float =
    if (position < 0f) -position.coerceAtLeast(-1f) * minimum.coerceAtMost(0f)
    else position.coerceAtMost(1f) * maximum.coerceAtLeast(0f)


internal data class ExpandedCardLayout(val button: Float, val controlGap: Float, val groupGap: Float,
    val readoutWidth: Float, val stacked: Boolean)

/** Allocate real, disjoint regions. Controls never shrink below a 48dp touch target. */
internal fun expandedCardLayout(width: Float, idealReadout: Float, minimumReadout: Float): ExpandedCardLayout {
    val controlGap = (width * .017f).coerceIn(4f, 8f)
    val groupGap = (width * .03f).coerceIn(8f, 14f)
    val button = ((width - idealReadout - groupGap - controlGap * 2) / 3).coerceIn(48f, 62f)
    val readout = width - 3 * button - 2 * controlGap - groupGap
    val stacked = readout < minimumReadout
    return ExpandedCardLayout(button, controlGap, groupGap, if (stacked) width else readout, stacked)
}


/** Overlap is intentional: do not also reserve the entire status-bar band inside the card. */
internal fun expandedContentClearance(position: app.expressiveclock.data.IslandExpandedPosition,
    statusBar: Float, cutoutBottom: Float, cardTop: Float): Float =
    if (position == app.expressiveclock.data.IslandExpandedPosition.OVERLAP_CUTOUT) 0f
    else (maxOf(statusBar, cutoutBottom) - cardTop - 16f).coerceAtLeast(0f)
```


---

## `app/src/main/java/app/expressiveclock/overlay/OverlayAccessibilityService.kt`

```kotlin
package app.expressiveclock.overlay

import android.accessibilityservice.AccessibilityService
import android.content.Intent
import android.view.accessibility.AccessibilityEvent

/**
 * Service used solely to host the Clock app's Dynamic Island overlay at the
 * TYPE_ACCESSIBILITY_OVERLAY system window layer above status bars.
 *
 * Observes window-change signals only; does not retrieve content, read screens, or perform gestures.
 */
class OverlayAccessibilityService : AccessibilityService() {

    override fun onServiceConnected() {
        super.onServiceConnected()
        instance = this
        OverlayManager.onAccessibilityServiceConnected(this)
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {
        OverlayManager.onForegroundWindowChanged()
    }

    override fun onConfigurationChanged(newConfig: android.content.res.Configuration) {
        super.onConfigurationChanged(newConfig)
        IslandDisplayMonitor.refresh()
    }

    override fun onInterrupt() {
        // Intentionally empty.
    }

    override fun onUnbind(intent: Intent?): Boolean {
        instance = null
        OverlayManager.onAccessibilityServiceDisconnected()
        return super.onUnbind(intent)
    }

    override fun onDestroy() {
        instance = null
        OverlayManager.onAccessibilityServiceDisconnected()
        super.onDestroy()
    }

    companion object {
        var instance: OverlayAccessibilityService? = null
            private set
    }
}
```


---

## `app/src/main/java/app/expressiveclock/overlay/OverlayManager.kt`

```kotlin
package app.expressiveclock.overlay

import app.expressiveclock.data.calibration
import app.expressiveclock.data.legacyCalibration
import androidx.compose.runtime.collectAsState
import android.content.Context
import androidx.compose.ui.graphics.graphicsLayer
import android.content.Intent
import android.graphics.PixelFormat
import android.graphics.Rect
import android.os.Build
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.util.Log
import android.view.View
import android.view.WindowInsets
import android.app.KeyguardManager
import android.view.Gravity
import android.view.MotionEvent
import android.view.WindowManager
import androidx.compose.foundation.clickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.ComposeView
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.core.content.getSystemService
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleRegistry
import androidx.lifecycle.ViewModelStore
import androidx.lifecycle.ViewModelStoreOwner
import androidx.lifecycle.setViewTreeLifecycleOwner
import androidx.lifecycle.setViewTreeViewModelStoreOwner
import androidx.savedstate.SavedStateRegistry
import androidx.savedstate.SavedStateRegistryController
import androidx.savedstate.SavedStateRegistryOwner
import androidx.savedstate.setViewTreeSavedStateRegistryOwner
import app.expressiveclock.MainActivity
import app.expressiveclock.alarm.ClockActionReceiver
import app.expressiveclock.alarm.Notifications
import app.expressiveclock.core.ClockTimer
import app.expressiveclock.core.Stopwatch
import app.expressiveclock.core.TimerState
import app.expressiveclock.data.ClockSettings
import app.expressiveclock.data.ClockStore
import app.expressiveclock.ui.theme.ClockTheme
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.launch
import java.time.Duration
import kotlin.math.roundToInt

/**
 * Controller for adding, updating, and removing the Dynamic Island overlay via WindowManager.
 *
 * Architecture:
 * - Layer 1 (Rendering Window): Full-screen MATCH_PARENT window with FLAG_NOT_TOUCHABLE for rendering smooth Compose animations.
 * - Layer 2 (Interaction Window): Bounded window matching activeIslandBounds for precise touch interaction.
 */
object OverlayManager {

    private var hiddenForFullscreen = false
    private val fullscreenHandler = Handler(Looper.getMainLooper())
    private var pendingFullscreenUpdate: Runnable? = null
    // Non-fitting accessibility windows can omit bar-change dispatches. Sample only
    // while an island window exists; no screen content or package names are inspected.
    private val fullscreenSampler = object : Runnable {
        override fun run() {
            val view = renderComposeView ?: return
            if (Build.VERSION.SDK_INT >= 30) view.rootWindowInsets?.let(::updateFullscreenVisibility)
            fullscreenHandler.postDelayed(this, 500)
        }
    }

    fun onForegroundWindowChanged() {
        if (Build.VERSION.SDK_INT >= 30) renderComposeView?.rootWindowInsets?.let(::updateFullscreenVisibility)
    }


    private fun updateFullscreenVisibility(insets: WindowInsets) {
        // Visibility, not inset size or window bounds, distinguishes immersive from edge-to-edge.
        // Older Android/OEM windows without reliable bar information keep the island visible.
        if (Build.VERSION.SDK_INT < 30) return
        val hide = fullscreenCandidate(insets)
        pendingFullscreenUpdate?.let(fullscreenHandler::removeCallbacks)
        pendingFullscreenUpdate = Runnable {
            if (hiddenForFullscreen != hide) {
                Log.d("OverlayManager", "Fullscreen suppression=$hide")
                hiddenForFullscreen = hide
                applyFullscreenVisibility()
            }
        }.also { fullscreenHandler.postDelayed(it, 160) }
    }

    @androidx.annotation.RequiresApi(30)
    private fun fullscreenCandidate(insets: WindowInsets): Boolean = runCatching {
        val observedInsets = windowManager?.currentWindowMetrics?.windowInsets ?: insets
        val knownStatusBar = observedInsets.getInsetsIgnoringVisibility(WindowInsets.Type.statusBars()).top > 0
        val locked = appContext?.getSystemService(KeyguardManager::class.java)?.isKeyguardLocked != false
        shouldHideIslandForFullscreen(
            if (knownStatusBar) observedInsets.isVisible(WindowInsets.Type.statusBars()) else null,
            if (knownStatusBar) observedInsets.isVisible(WindowInsets.Type.navigationBars()) else null,
            locked, isCalibrationPreviewing)
    }.getOrDefault(false)

    private fun applyFullscreenVisibility() {
        val visibility = if (hiddenForFullscreen && !isCalibrationPreviewing) View.INVISIBLE else View.VISIBLE
        renderComposeView?.visibility = visibility
        interactionComposeView?.visibility = visibility
    }

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    private var windowManager: WindowManager? = null
    private var overlayWindowManager: WindowManager? = null
    private var renderComposeView: ComposeView? = null
    private var interactionComposeView: ComposeView? = null
    private var overlayLifecycleOwner: OverlayLifecycleOwner? = null
    private var overlayViewModelStoreOwner: ViewModelStoreOwner? = null
    private var appContext: Context? = null

    private var currentSettings: ClockSettings? = null
    // DataStore emissions can lag behind a drag. Never replay those older values
    // over the immediate calibration preview.
    private var calibrationPreviewSettings: ClockSettings? = null
    private var currentTimer: ClockTimer? = null
    private var currentStopwatch: Stopwatch? = null

    // Persistent observable Compose states for the single composition tree
    private var overlayTimerState by mutableStateOf<ClockTimer?>(null)
    private var overlayStopwatchState by mutableStateOf<Stopwatch?>(null)
    private var overlaySettingsState by mutableStateOf(ClockSettings())
    private var overlayIsExitingState by mutableStateOf(false)
    private var overlayIsCalibrationState by mutableStateOf(false)
    private var overlayIsExpandedState by mutableStateOf(false)
    private var expandedClearanceState by androidx.compose.runtime.mutableFloatStateOf(0f)

    // Dedicated exit request ID and visual snapshot states
    private var overlayExitRequestId by mutableStateOf(0)
    private var overlayExitTimerState: ClockTimer? by mutableStateOf(null)
    private var overlayExitStopwatchState: Stopwatch? by mutableStateOf(null)

    private var isOverlayAttached = false
    private var isOverlayBeingRemoved = false
    private var isCalibrationPreviewing = false
    private var activeIslandBounds: Rect? = null
    private var overlayGeneration = 0

    // Queued app launch state to prevent foreground lifecycle crashes
    private var pendingAppLaunchTab: String? = null
    private var pendingExitAfterCollapse = false

    fun init(context: Context) {
        if (appContext != null) return
        val app = context.applicationContext
        appContext = app
        IslandDisplayMonitor.start(app)
        windowManager = app.getSystemService<WindowManager>()

        scope.launch { IslandDisplayMonitor.state.collect { syncOverlayState() } }
        val store = ClockStore(app)
        scope.launch {
            combine(store.settings, store.timer, store.stopwatch) { s, t, sw ->
                Triple(s, t, sw)
            }.collect { (s, t, sw) ->
                currentSettings = s
                currentTimer = t
                currentStopwatch = sw
                syncOverlayState()
            }
        }
    }

    fun onAccessibilityServiceConnected(service: OverlayAccessibilityService) {
        val wm = service.getSystemService(Context.WINDOW_SERVICE) as? WindowManager
        if (wm != null) {
            windowManager = wm
            Log.d("DynamicIslandOverlay", "onAccessibilityServiceConnected: windowManager set (wmHash=${System.identityHashCode(wm)})")
        }
        syncOverlayState()
    }

    fun onAccessibilityServiceDisconnected() {
        Log.d("DynamicIslandOverlay", "onAccessibilityServiceDisconnected: service disconnected (overlayAttached=$isOverlayAttached, force=true)")
        if (isOverlayAttached) {
            removeOverlayNow(force = true)
        }
        windowManager = null
    }

    fun showCalibrationPreview(context: Context, settings: ClockSettings) {
        calibrationPreviewSettings = settings
        isCalibrationPreviewing = true
        syncOverlayState()
    }

    fun updateCalibrationPreview(settings: ClockSettings) {
        calibrationPreviewSettings = settings
        overlaySettingsState = settings
        // The preview already occupies a full-screen window. Moving its Compose
        // content does not require relaying out the Android overlay window.
        if (!isOverlayAttached) {
            syncOverlayState()
        }
    }

    fun hideCalibrationPreview() {
        isCalibrationPreviewing = false
        calibrationPreviewSettings = null
        overlayIsCalibrationState = false
        syncOverlayState()
    }

    fun onClockActivityForegrounded() {
        syncOverlayState()
    }

    fun hideImmediatelyForAppForeground() {
        if (!isOverlayAttached) return
        Log.d("DynamicIslandOverlay", "OVERLAY DETACH START (gen=$overlayGeneration, wmHash=${System.identityHashCode(overlayWindowManager)})")
        removeOverlayNow(force = true)
        Log.d("DynamicIslandOverlay", "OVERLAY DETACH COMPLETE")
    }

    fun onAppForegroundChanged(inForeground: Boolean) {
        syncOverlayState()
    }

    fun requestOpenAppAfterAnimatedExit(tab: String) {
        Log.d("DynamicIslandOverlay", "DI APP OPEN REQUESTED tab=$tab")
        val context = appContext ?: return
        launchMainActivityNow(context, tab)
        if (overlayIsExpandedState) {
            overlayIsExpandedState = false
            updateWindowParams()
        }
    }

    private fun launchMainActivityNow(context: Context, tab: String) {
        Log.d("DynamicIslandOverlay", "DI LAUNCHING MAIN ACTIVITY tab=$tab")
        runCatching {
            val intent = Intent(context, MainActivity::class.java).apply {
                flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_CLEAR_TOP
                putExtra(Notifications.EXTRA_TAB, tab)
            }
            context.startActivity(intent)
        }.onFailure { Log.e("OverlayManager", "Failed to launch MainActivity", it) }
    }

    fun syncOverlayState() {
        val context = appContext ?: return

        // Stored state and the accessibility window are sufficient after a service restart;
        // waiting for an Activity would leave a running background timer without an island.
        val settings = (if (isCalibrationPreviewing) calibrationPreviewSettings else currentSettings) ?: return
        val isEnabledInSettings = settings.theme.dynamicIsland && settings.theme.calibration(IslandDisplayMonitor.state.value.calibrationProfile).enabled
        val isAccConnected = OverlayAccessibilityService.instance != null

        overlaySettingsState = settings
        overlayIsCalibrationState = isCalibrationPreviewing

        val timer = currentTimer ?: if (isCalibrationPreviewing) ClockTimer(id = -1, label = "Preview", total = Duration.ofMinutes(5), state = TimerState.RUNNING) else null
        val sw = currentStopwatch

        val isTimerActive = timer != null && timer.state != TimerState.FINISHED
        val isStopwatchActive = sw != null && (sw.running || !sw.accumulated.isZero)
        val shouldShow = isCalibrationPreviewing || ((isTimerActive || isStopwatchActive) && isEnabledInSettings && isAccConnected)

        Log.d("DynamicIslandOverlay", "DI SYNC STATE: shouldShow=$shouldShow, isTimerActive=$isTimerActive, isStopwatchActive=$isStopwatchActive, isAttached=$isOverlayAttached, isExiting=$overlayIsExitingState")

        if (shouldShow) {
            if (overlayIsExitingState && !isCalibrationPreviewing) {
                Log.d("DynamicIslandOverlay", "syncOverlayState: Ignoring show request because animated exit is in progress")
                return
            }
            overlayTimerState = timer
            overlayStopwatchState = sw
            overlayIsExitingState = false
            overlayExitRequestId = 0
            // The AccessibilityService hosts the overlay window. Do not use a mediaPlayback
            // foreground service just to keep this overlay process alive.
            showOrUpdateOverlay(context)
        } else {
            if (isOverlayAttached && !overlayIsExitingState) {
                Log.d("DynamicIslandOverlay", "DI HIDE ROUTE = ANIMATED_EXIT")
                requestAnimatedOverlayHide()
            } else if (!isOverlayAttached) {
                overlayTimerState = timer
                overlayStopwatchState = sw
                removeOverlayNow(force = false)
            }
        }
    }

    fun requestAnimatedOverlayHide() {
        if (!isOverlayAttached || overlayIsExitingState) return

        if (overlayIsExpandedState) {
            Log.d("DynamicIslandOverlay", "DI COLLAPSE BEFORE CAMERA EXIT")
            pendingExitAfterCollapse = true

            val interactionView = interactionComposeView
            val wm = overlayWindowManager ?: windowManager
            if (interactionView != null && wm != null) {
                interactionComposeView = null
                runCatching {
                    wm.removeViewImmediate(interactionView)
                }
            }

            overlayIsExpandedState = false
            updateWindowParams()
            return
        }

        overlayExitRequestId++
        overlayExitTimerState = overlayTimerState
        overlayExitStopwatchState = overlayStopwatchState
        overlayIsExitingState = true
        Log.d("DynamicIslandOverlay", "DI BEGIN ANIMATED EXIT request=$overlayExitRequestId")
        Log.d("DynamicIslandOverlay", "DI EXIT SNAPSHOT timer=$overlayExitTimerState stopwatch=$overlayExitStopwatchState")

        // Remove Layer 2 interaction window immediately so user cannot touch the shrinking island
        val interactionView = interactionComposeView
        val wm = overlayWindowManager ?: windowManager
        if (interactionView != null && wm != null) {
            interactionComposeView = null
            runCatching {
                wm.removeViewImmediate(interactionView)
            }.onFailure { Log.e("OverlayManager", "Failed to remove interaction view during exit start", it) }
        }
    }

    private fun triggerExitAnimation() {
        requestAnimatedOverlayHide()
    }

    private fun showOrUpdateOverlay(context: Context) {
        val service = OverlayAccessibilityService.instance
        if (service == null && !isCalibrationPreviewing) {
            removeOverlayNow(force = true)
            return
        }
        val wm = windowManager ?: return

        if (renderComposeView == null) {
            overlayGeneration++
            val currentGen = overlayGeneration

            val lifecycleOwner = OverlayLifecycleOwner()
            lifecycleOwner.performRestore(null)
            lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
            lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_START)
            lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_RESUME)
            overlayLifecycleOwner = lifecycleOwner

            val viewModelStoreOwner = object : ViewModelStoreOwner {
                override val viewModelStore: ViewModelStore = ViewModelStore()
            }
            overlayViewModelStoreOwner = viewModelStoreOwner

            // Layer 1: Rendering View (Full-screen, non-touchable)
            val renderView = ComposeView(context).apply {
                setViewTreeLifecycleOwner(lifecycleOwner)
                setViewTreeViewModelStoreOwner(viewModelStoreOwner)
                setViewTreeSavedStateRegistryOwner(lifecycleOwner)

                setOnApplyWindowInsetsListener { _, insets ->
                    updateFullscreenVisibility(insets)
                    insets
                }
                setContent {
                    PersistentOverlayContent(context, currentGen)
                }
            }
            hiddenForFullscreen = if (Build.VERSION.SDK_INT >= 30)
                fullscreenCandidate(wm.currentWindowMetrics.windowInsets) else false
            renderComposeView = renderView
            applyFullscreenVisibility()

            val renderParams = createRenderParams() ?: return
            val addResult = runCatching {
                wm.addView(renderView, renderParams)
                // Compose installs its own inset listener during attachment. Observe after
                // that setup, returning the original insets to preserve Compose's handling.
                renderView.post {
                    if (renderComposeView === renderView) {
                        renderView.setOnApplyWindowInsetsListener { _, insets ->
                            updateFullscreenVisibility(insets)
                            insets
                        }
                        if (Build.VERSION.SDK_INT >= 30) renderView.setWindowInsetsAnimationCallback(
                            object : android.view.WindowInsetsAnimation.Callback(DISPATCH_MODE_CONTINUE_ON_SUBTREE) {
                                override fun onProgress(insets: WindowInsets, runningAnimations: MutableList<android.view.WindowInsetsAnimation>): WindowInsets {
                                    updateFullscreenVisibility(insets)
                                    return insets
                                }
                            })
                        renderView.rootWindowInsets?.let(::updateFullscreenVisibility)
                        renderView.requestApplyInsets()
                        fullscreenHandler.removeCallbacks(fullscreenSampler)
                        fullscreenHandler.post(fullscreenSampler)
                    }
                }
                isOverlayAttached = true
                overlayWindowManager = wm
            }
            if (addResult.isFailure) {
                Log.e("OverlayManager", "Failed to add render overlay window (gen=$currentGen, wmHash=${System.identityHashCode(wm)})", addResult.exceptionOrNull())
                renderComposeView = null
                isOverlayAttached = false
                overlayWindowManager = null
                activeIslandBounds = null
            } else {
                Log.d("DynamicIslandOverlay", "Render overlay window added successfully (gen=$currentGen, wmHash=${System.identityHashCode(wm)})")
            }
        } else if (!isCalibrationPreviewing) {
            updateWindowParams()
        }

        applyFullscreenVisibility()

        // Layer 2: Interaction Window (Bounded, touchable)
        if (isOverlayAttached && !isCalibrationPreviewing) {
            val attachedWm = overlayWindowManager ?: wm
            showOrUpdateInteractionView(context, attachedWm, overlayGeneration)
        } else if (isCalibrationPreviewing && interactionComposeView != null) {
            val attachedWm = overlayWindowManager ?: wm
            runCatching {
                attachedWm.removeViewImmediate(interactionComposeView)
            }
            interactionComposeView = null
        }
    }

    private fun showOrUpdateInteractionView(context: Context, wm: WindowManager, generation: Int) {
        val attachedWm = overlayWindowManager ?: wm
        if (!isOverlayAttached || generation != overlayGeneration || overlayIsExitingState) return
        val bounds = activeIslandBounds ?: return
        if (bounds.width() <= 0 || bounds.height() <= 0) return

        val params = createInteractionParams(bounds) ?: return

        if (interactionComposeView == null) {
            val lifecycleOwner = overlayLifecycleOwner ?: OverlayLifecycleOwner().apply {
                performRestore(null)
                handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
                handleLifecycleEvent(Lifecycle.Event.ON_START)
                handleLifecycleEvent(Lifecycle.Event.ON_RESUME)
            }
            val viewModelStoreOwner = overlayViewModelStoreOwner ?: object : ViewModelStoreOwner {
                override val viewModelStore: ViewModelStore = ViewModelStore()
            }

            val view = ComposeView(context).apply {
                setViewTreeLifecycleOwner(lifecycleOwner)
                setViewTreeViewModelStoreOwner(viewModelStoreOwner)
                setViewTreeSavedStateRegistryOwner(lifecycleOwner)

                setOnTouchListener { _, event ->
                    if (event.actionMasked == MotionEvent.ACTION_OUTSIDE && overlayIsExpandedState) {
                        overlayIsExpandedState = false
                        updateWindowParams()
                    }
                    false
                }

                setContent {
                    InteractionOverlayContent(context)
                }
            }
            interactionComposeView = view
            applyFullscreenVisibility()

            val addResult = runCatching {
                attachedWm.addView(view, params)
            }
            if (addResult.isFailure) {
                Log.e("OverlayManager", "Failed to add interaction window (gen=$generation, wmHash=${System.identityHashCode(attachedWm)})", addResult.exceptionOrNull())
                interactionComposeView = null
            } else {
                Log.d("DynamicIslandOverlay", "Interaction window added successfully (gen=$generation, wmHash=${System.identityHashCode(attachedWm)})")
            }
        } else {
            val updateResult = runCatching {
                attachedWm.updateViewLayout(interactionComposeView, params)
            }
            if (updateResult.isFailure) {
                Log.e("OverlayManager", "Failed to update interaction window params (gen=$generation, wmHash=${System.identityHashCode(attachedWm)})", updateResult.exceptionOrNull())
            }
        }
    }

    @Composable
    private fun PersistentOverlayContent(context: Context, generation: Int) {
        val settings = overlaySettingsState
        val visualTimer = if (overlayIsExitingState) overlayExitTimerState else overlayTimerState
        val visualStopwatch = if (overlayIsExitingState) overlayExitStopwatchState else overlayStopwatchState

        ClockTheme {
            val display by IslandDisplayMonitor.state.collectAsState()
            val calibration = settings.theme.calibration(display.calibrationProfile)
            DynamicIslandOverlayContent(
                timer = visualTimer,
                stopwatch = visualStopwatch,
                exitRequestId = overlayExitRequestId,
                topInsetDp = 8.dp,
                cutoutWidthDp = 48.dp,
                alignment = calibration.alignment,
                expandedPosition = calibration.expandedPosition,
                offsetXDp = calibration.x.dp,
                offsetYDp = calibration.y.dp,
                widthOffsetDp = calibration.width.dp,
                heightOffsetDp = calibration.height.dp,
                isCalibrationPreview = overlayIsCalibrationState,
                isExiting = overlayIsExitingState,
                isExpanded = overlayIsExpandedState,
                onExpandedClearanceChanged = { expandedClearanceState = it },
                onBoundsChanged = { bounds ->
                    if (isOverlayAttached && generation == overlayGeneration) {
                        activeIslandBounds = bounds
                        val attachedWm = overlayWindowManager ?: windowManager
                        if (attachedWm != null && !isCalibrationPreviewing) {
                            showOrUpdateInteractionView(context, attachedWm, generation)
                        }
                    }
                },
                onCollapsedSettled = {
                    if (generation == overlayGeneration && pendingExitAfterCollapse) {
                        Log.d("DynamicIslandOverlay", "DI COLLAPSE COMPLETE")
                        pendingExitAfterCollapse = false
                        requestAnimatedOverlayHide()
                    }
                },
                onExitComplete = {
                    if (generation == overlayGeneration) {
                        Log.d("DynamicIslandOverlay", "DI EXIT COMPLETE -> REMOVING (force=true, gen=$generation)")
                        Handler(Looper.getMainLooper()).post {
                            if (generation == overlayGeneration) {
                                removeOverlayNow(force = true)
                            }
                        }
                    }
                },
                onExpandChanged = { expanded ->
                    if (generation == overlayGeneration) {
                        overlayIsExpandedState = expanded
                        updateWindowParams()
                    }
                },
                onOpenAppTab = { tab ->
                    if (generation == overlayGeneration) {
                        requestOpenAppAfterAnimatedExit(tab)
                    }
                },
                onPauseResumeTimer = { sendAction(context, ClockActionReceiver.ACTION_TIMER_TOGGLE) },
                onAddTenTimer = { sendAction(context, ClockActionReceiver.ACTION_TIMER_ADD) },
                onCancelTimer = { sendAction(context, if (visualTimer != null) ClockActionReceiver.ACTION_TIMER_CANCEL else ClockActionReceiver.ACTION_SW_RESET) },
                onToggleStopwatch = { sendAction(context, ClockActionReceiver.ACTION_SW_TOGGLE) },
                onLapStopwatch = { sendAction(context, ClockActionReceiver.ACTION_SW_LAP) },
            )
        }
    }

    @Composable
    private fun InteractionOverlayContent(context: Context) {
        val isTimerActive = overlayTimerState != null && overlayTimerState?.state != TimerState.FINISHED
        ClockTheme {
            Box(Modifier.fillMaxSize().clickable(indication = null,
                interactionSource = remember { MutableInteractionSource() }) {
                if (overlayIsExpandedState) requestOpenAppAfterAnimatedExit(
                    if (isTimerActive) Notifications.TAB_TIMERS else Notifications.TAB_STOPWATCH)
                else { overlayIsExpandedState = true; updateWindowParams() }
            }) {
                if (overlayIsExpandedState) {
                    // Share the visual card's measurements instead of maintaining drifting hit boxes.
                    Box(Modifier.fillMaxSize().padding(start = 20.dp, end = 20.dp, top = 16.dp + expandedClearanceState.dp, bottom = 16.dp)
                        .graphicsLayer { alpha = 0f }) {
                        ExpandedIslandCard(
                            timer = overlayTimerState, stopwatch = overlayStopwatchState,
                            onOpenTimerTab = { requestOpenAppAfterAnimatedExit(Notifications.TAB_TIMERS) },
                            onOpenStopwatchTab = { requestOpenAppAfterAnimatedExit(Notifications.TAB_STOPWATCH) },
                            onPauseResumeTimer = { sendAction(context, ClockActionReceiver.ACTION_TIMER_TOGGLE) },
                            onAddTenTimer = { sendAction(context, ClockActionReceiver.ACTION_TIMER_ADD) },
                            onCancelTimer = { sendAction(context, if (isTimerActive) ClockActionReceiver.ACTION_TIMER_CANCEL else ClockActionReceiver.ACTION_SW_RESET) },
                            onToggleStopwatch = { sendAction(context, ClockActionReceiver.ACTION_SW_TOGGLE) },
                            onLapStopwatch = { sendAction(context, ClockActionReceiver.ACTION_SW_LAP) },
                        )
                    }
                }
            }
        }
    }

    private fun updateWindowParams() {
        val view = renderComposeView ?: return
        val wm = overlayWindowManager ?: windowManager ?: return
        if (!isOverlayAttached) return
        val params = createRenderParams() ?: return
        runCatching {
            wm.updateViewLayout(view, params)
        }.onFailure { Log.e("OverlayManager", "Failed to update render window params (wmHash=${System.identityHashCode(wm)})", it) }

        if (interactionComposeView != null) {
            val interactionParams = activeIslandBounds?.let { createInteractionParams(it) }
            if (interactionParams != null) {
                runCatching {
                    wm.updateViewLayout(interactionComposeView, interactionParams)
                }.onFailure { Log.e("OverlayManager", "Failed to update interaction window params (wmHash=${System.identityHashCode(wm)})", it) }
            }
        }
    }

    private fun createRenderParams(): WindowManager.LayoutParams? {
        val service = OverlayAccessibilityService.instance
        if (service == null && !isCalibrationPreviewing) {
            return null
        }

        // FLAG_NOT_TOUCHABLE guarantees Layer 1 never blocks touches outside the island!
        val touchFlags = WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE or
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
            WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN

        return WindowManager.LayoutParams(
            WindowManager.LayoutParams.MATCH_PARENT,
            WindowManager.LayoutParams.MATCH_PARENT,
            WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
            touchFlags,
            PixelFormat.TRANSLUCENT,
        ).apply {
            gravity = Gravity.TOP or Gravity.CENTER_HORIZONTAL
            x = 0
            y = 0
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                setFitInsetsTypes(0)
                layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS
            } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                @Suppress("DEPRECATION")
                layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES
            }
        }
    }

    private fun createInteractionParams(bounds: Rect): WindowManager.LayoutParams? {
        val service = OverlayAccessibilityService.instance
        if (service == null && !isCalibrationPreviewing) {
            return null
        }

        val touchFlags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
            WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
            WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
            WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN

        return WindowManager.LayoutParams(
            bounds.width().coerceAtLeast(100),
            bounds.height().coerceAtLeast(40),
            WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
            touchFlags,
            PixelFormat.TRANSLUCENT,
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = bounds.left
            y = bounds.top
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                setFitInsetsTypes(0)
                layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS
            } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                @Suppress("DEPRECATION")
                layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES
            }
        }
    }

    private fun removeOverlayNow(force: Boolean = false) {
        fullscreenHandler.removeCallbacks(fullscreenSampler)
        pendingFullscreenUpdate?.let(fullscreenHandler::removeCallbacks)
        if (isOverlayBeingRemoved) return

        if (overlayIsExitingState && !force) {
            Log.d("DynamicIslandOverlay", "DI REMOVE IGNORED force=false while overlayIsExitingState=true (gen=$overlayGeneration)")
            return
        }

        Log.d("DynamicIslandOverlay", "DI REMOVE force=$force exiting=$overlayIsExitingState (gen=$overlayGeneration)")
        isOverlayBeingRemoved = true

        val pendingTab = pendingAppLaunchTab
        pendingAppLaunchTab = null
        pendingExitAfterCollapse = false
        overlayExitTimerState = null
        overlayExitStopwatchState = null
        overlayExitRequestId = 0

        overlayGeneration++
        val renderView = renderComposeView
        val interactionView = interactionComposeView
        val wm = overlayWindowManager ?: windowManager
        val lifecycleOwner = overlayLifecycleOwner
        val vmStoreOwner = overlayViewModelStoreOwner

        renderComposeView = null
        interactionComposeView = null
        overlayLifecycleOwner = null
        overlayViewModelStoreOwner = null
        overlayWindowManager = null
        isOverlayAttached = false
        overlayIsExpandedState = false
        overlayIsExitingState = false
        activeIslandBounds = null

        try {
            if (interactionView != null && wm != null) {
                val removeResult = runCatching {
                    wm.removeViewImmediate(interactionView)
                }
                if (removeResult.isFailure) {
                    Log.e("OverlayManager", "Failed to remove interaction view (wmHash=${System.identityHashCode(wm)})", removeResult.exceptionOrNull())
                } else {
                    Log.d("DynamicIslandOverlay", "DI INTERACTION WINDOW REMOVED")
                }

                runCatching {
                    interactionView.disposeComposition()
                }.onFailure { Log.e("OverlayManager", "Failed to dispose interaction composition", it) }
            }

            if (renderView != null && wm != null) {
                val removeResult = runCatching {
                    wm.removeViewImmediate(renderView)
                }
                if (removeResult.isFailure) {
                    Log.e("OverlayManager", "Failed to remove render view (wmHash=${System.identityHashCode(wm)})", removeResult.exceptionOrNull())
                } else {
                    Log.d("DynamicIslandOverlay", "DI RENDER WINDOW REMOVED")
                }

                runCatching {
                    renderView.disposeComposition()
                }.onFailure { Log.e("OverlayManager", "Failed to dispose render composition", it) }
            }

            if (lifecycleOwner != null) {
                runCatching {
                    lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE)
                    lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_STOP)
                    lifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY)
                }.onFailure { Log.e("OverlayManager", "Failed to destroy overlay lifecycle", it) }
            }

            if (vmStoreOwner != null) {
                runCatching {
                    vmStoreOwner.viewModelStore.clear()
                }.onFailure { Log.e("OverlayManager", "Failed to clear overlay view model store", it) }
            }

            Log.d("DynamicIslandOverlay", "DI COMPOSITIONS DISPOSED & OVERLAY DETACH COMPLETE")

            if (pendingTab != null) {
                Handler(Looper.getMainLooper()).post {
                    val ctx = appContext ?: return@post
                    launchMainActivityNow(ctx, pendingTab)
                }
            }


        } finally {
            isOverlayBeingRemoved = false
        }
    }

    fun removeOverlay() {
        triggerExitAnimation()
    }

    private fun sendAction(context: Context, action: String) {
        val intent = Intent(context, ClockActionReceiver::class.java).setAction(action)
        context.sendBroadcast(intent)
    }

    private class OverlayLifecycleOwner : SavedStateRegistryOwner {
        private val lifecycleRegistry = LifecycleRegistry(this)
        private val savedStateRegistryController = SavedStateRegistryController.create(this)

        override val lifecycle: Lifecycle get() = lifecycleRegistry
        override val savedStateRegistry: SavedStateRegistry
            get() = savedStateRegistryController.savedStateRegistry

        fun performRestore(savedState: Bundle?) {
            savedStateRegistryController.performRestore(savedState)
        }

        fun handleLifecycleEvent(event: Lifecycle.Event) {
            lifecycleRegistry.handleLifecycleEvent(event)
        }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/ClockDock.kt`

```kotlin
package app.expressiveclock.ui

import androidx.compose.animation.core.Animatable
import androidx.compose.animation.core.animateDpAsState
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.core.spring
import androidx.compose.animation.core.tween
import androidx.compose.foundation.background
import androidx.compose.foundation.selection.selectable
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.onClick
import androidx.compose.ui.semantics.role
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxWithConstraints
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.RowScope
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.widthIn
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.offset
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.rounded.Add
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.luminance
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.semantics.clearAndSetSemantics
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.IntOffset

/**
 * Fixed-stage dock. The add button never changes the measured width of the stage, so moving from
 * Alarms to another tab cannot make the dock jump/stutter. Instead the dock and add button animate
 * within the same centered stage.
 */
@Composable
fun ClockDock(
    destinations: List<Tab>,
    selected: Tab,
    onSelect: (Tab) -> Unit,
    showAdd: Boolean,
    addLabel: String,
    onAdd: () -> Unit,
    modifier: Modifier = Modifier,
    enableGlow: Boolean = true,
    pagePosition: (() -> Float)? = null,
) {
    val dark = MaterialTheme.colorScheme.surface.luminance() < 0.5f
    val blue = MaterialTheme.colorScheme.primary
    val dockTint = if (dark) MaterialTheme.colorScheme.surfaceContainer.copy(alpha = .88f) else MaterialTheme.colorScheme.surfaceContainerLow.copy(alpha = .84f)

    val addAlpha by animateFloatAsState(
        targetValue = if (showAdd) 1f else 0f,
        animationSpec = tween(180),
        label = "addAlpha",
    )
    val addScale by animateFloatAsState(
        targetValue = if (showAdd) 1f else .78f,
        animationSpec = spring(dampingRatio = 0.9f, stiffness = 620f),
        label = "addScale",
    )

    BoxWithConstraints(
        modifier = modifier.widthIn(max = DOCK_STAGE_WIDTH + 16.dp).fillMaxWidth().padding(horizontal = 8.dp),
        contentAlignment = Alignment.Center,
    ) {
        val addSize = minOf(DOCK_HEIGHT, (maxWidth - 218.dp).coerceAtLeast(48.dp))
        val dockWidth = minOf(DOCK_WIDTH, (maxWidth - addSize - FAB_GAP).coerceAtLeast(0.dp))
        val dockShift by animateDpAsState(
            targetValue = if (showAdd) 0.dp else (addSize + FAB_GAP) / 2,
            animationSpec = spring(dampingRatio = 0.86f, stiffness = 560f),
            label = "dockShift",
        )

        Row(
            modifier = Modifier
                .width(dockWidth + FAB_GAP + addSize)
                .padding(horizontal = 0.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(FAB_GAP),
        ) {
            Box(
                modifier = Modifier
                    .width(dockWidth)
                    .height(DOCK_HEIGHT)
                    .graphicsLayer { translationX = dockShift.toPx() }
                    .luminousElasticClickable(
                        shape = RoundedCornerShape(DOCK_HEIGHT / 2),
                        maxStretchPercent = 0.06f,
                        maxTranslateDp = 1.8.dp,
                    )
                    .glass(
                        RoundedCornerShape(DOCK_HEIGHT / 2),
                        dockTint,
                        .88f,
                        shadowElevation = if (enableGlow) 18.dp else 0.dp,
                    )
                    .then(if (enableGlow) Modifier.blueGlow(RoundedCornerShape(DOCK_HEIGHT / 2)) else Modifier),
                contentAlignment = Alignment.Center,
            ) {
                BoxWithConstraints(Modifier.fillMaxSize().padding(horizontal = 8.dp)) {
                    val selectedIndex = destinations.indexOf(selected).coerceAtLeast(0)
                    val itemWidth = maxWidth / destinations.size
                    val indicatorSize = minOf(50.dp, ((dockWidth - 16.dp) / 4 - 4.dp).coerceAtLeast(28.dp))
                    val selectedOffset = if (pagePosition == null) animateDpAsState(
                        targetValue = itemWidth * selectedIndex + (itemWidth - indicatorSize) / 2,
                        animationSpec = spring(dampingRatio = 0.84f, stiffness = 600f),
                        label = "dockIndicator",
                    ) else null
                    Box(
                        modifier = Modifier
                            .offset {
                                // Read continuous pager progress during placement, not
                                // composition. No extra spring trails the user's finger.
                                val position = pagePosition?.invoke()?.coerceIn(0f, destinations.lastIndex.toFloat())
                                val x = if (position == null) selectedOffset!!.value
                                    else itemWidth * position + (itemWidth - indicatorSize) / 2
                                IntOffset(x.roundToPx(), ((DOCK_HEIGHT - indicatorSize) / 2).roundToPx())
                            }
                            .size(indicatorSize)
                            .clip(CircleShape)
                            .background(
                                if (dark) Color(0xFF24303B) else Color.White.copy(alpha = .99f)
                            ),
                    )
                    Row(
                        modifier = Modifier.fillMaxSize(),
                        horizontalArrangement = Arrangement.SpaceEvenly,
                        verticalAlignment = Alignment.CenterVertically,
                    ) {
                        destinations.forEach { d ->
                            DockItem(
                                tab = d,
                                selected = d == selected,
                                iconColor = if (dark) Color.White.copy(alpha = .94f) else blue,
                                onClick = { onSelect(d) },
                            )
                        }
                    }
                }
            }

            Box(
                modifier = Modifier
                    .size(addSize)
                    .graphicsLayer {
                        alpha = addAlpha
                        scaleX = addScale
                        scaleY = addScale
                    }
                    .luminousElasticClickable(
                        enabled = showAdd,
                        shape = CircleShape,
                        maxStretchPercent = 0.08f,
                        maxTranslateDp = 2.0.dp,
                        onClick = onAdd,
                    )
                    .clip(CircleShape)
                    .glass(
                        CircleShape,
                        if (dark) MaterialTheme.colorScheme.primaryContainer.copy(alpha = .72f)
                        else MaterialTheme.colorScheme.primaryContainer.copy(alpha = .58f),
                        .92f,
                        shadowElevation = if (enableGlow) 18.dp else 0.dp,
                    )
                    .then(if (enableGlow) Modifier.blueGlow(CircleShape) else Modifier)
                    .clearAndSetSemantics {
                        if (showAdd) {
                            contentDescription = addLabel
                            role = Role.Button
                            onClick(label = addLabel) { onAdd(); true }
                        }
                    },
                contentAlignment = Alignment.Center,
            ) {
                Icon(Icons.Rounded.Add, contentDescription = null, tint = MaterialTheme.colorScheme.onPrimaryContainer, modifier = Modifier.size(31.dp))
            }
        }
    }
}

@Composable
private fun RowScope.DockItem(tab: Tab, selected: Boolean, iconColor: Color, onClick: () -> Unit) {
    var tapCount by remember { mutableIntStateOf(0) }
    var previouslySelected by remember { mutableStateOf(selected) }
    // A swipe selects a destination without invoking onClick. Use the same icon
    // animation for every new selection, without playing it on initial composition.
    LaunchedEffect(selected) {
        if (selected && !previouslySelected) tapCount++
        previouslySelected = selected
    }
    val rotation = remember { Animatable(0f) }
    val selectedScale by animateFloatAsState(
        targetValue = if (selected) 1f else .94f,
        animationSpec = tween(180),
        label = "dockScale",
    )

    LaunchedEffect(tapCount) {
        if (tapCount == 0) return@LaunchedEffect
        if (tab == Tab.TIMERS) {
            if (android.animation.ValueAnimator.areAnimatorsEnabled()) {
                val target = ((rotation.value / 180f).toInt() + 1) * 180f
                rotation.animateTo(target, tween(320))
            }
            rotation.snapTo(0f)
            return@LaunchedEffect
        }
        if (tab == Tab.WORLD || tab == Tab.STOPWATCH) {
            if (android.animation.ValueAnimator.areAnimatorsEnabled()) {
                val target = ((rotation.value / 360f).toInt() + 1) * 360f
                rotation.animateTo(target, tween(if (tab == Tab.WORLD) 600 else 420,
                    easing = androidx.compose.animation.core.FastOutSlowInEasing))
            }
            rotation.snapTo(0f)
            return@LaunchedEffect
        }
        rotation.snapTo(0f)
        when (tab) {
            Tab.ALARMS -> {
                rotation.animateTo(-5f, tween(65))
                rotation.animateTo(5f, tween(85))
                rotation.animateTo(-3f, tween(65))
                rotation.animateTo(0f, tween(65))
            }
            Tab.WORLD -> Unit
            Tab.TIMERS -> Unit
            Tab.STOPWATCH -> Unit
        }
        rotation.snapTo(0f)
    }

    Box(
        modifier = Modifier
            .weight(1f)
            .height(DOCK_HEIGHT)
            .selectable(
                selected = selected,
                role = Role.Tab,
                interactionSource = remember { MutableInteractionSource() },
                indication = null,
                onClick = {
                    // New selections animate via selected above. A repeated tap
                    // still replays the animation without double-triggering a new tab.
                    if (selected) tapCount++
                    onClick()
                },
            ),
        contentAlignment = Alignment.Center,
    ) {
        if (tab == Tab.WORLD) {
            RotatingGlobe(
                phase = { rotation.value },
                tint = if (selected) MaterialTheme.colorScheme.onSurface else iconColor,
                modifier = Modifier.size(27.dp).graphicsLayer { scaleX = selectedScale; scaleY = selectedScale }
                    .clearAndSetSemantics { contentDescription = tab.label },
            )
        } else if (tab == Tab.STOPWATCH) {
            StopwatchDockIcon(
                phase = { rotation.value },
                tint = if (selected) MaterialTheme.colorScheme.onSurface else iconColor,
                modifier = Modifier.size(27.dp).graphicsLayer { scaleX = selectedScale; scaleY = selectedScale }
                    .clearAndSetSemantics { contentDescription = tab.label },
            )
        } else {
        Icon(
            imageVector = tab.icon,
            contentDescription = tab.label,
            tint = if (selected) MaterialTheme.colorScheme.onSurface else iconColor,
            modifier = Modifier
                .size(27.dp)
                .graphicsLayer {
                    scaleX = selectedScale
                    scaleY = selectedScale
                    when (tab) {
                        Tab.WORLD -> rotationZ = rotation.value
                        Tab.ALARMS -> rotationZ = rotation.value
                        Tab.TIMERS -> rotationZ = rotation.value
                        Tab.STOPWATCH -> Unit
                    }
                    cameraDistance = 18f * density
                },
        )
        }
    }
}

val DOCK_HEIGHT = 72.dp
private val DOCK_WIDTH = 252.dp
private val FAB_GAP = 10.dp
private val DOCK_STAGE_WIDTH = DOCK_WIDTH + FAB_GAP + DOCK_HEIGHT
```


---

## `app/src/main/java/app/expressiveclock/ui/Glass.kt`

```kotlin
package app.expressiveclock.ui

import androidx.compose.animation.core.Animatable
import androidx.compose.animation.core.FastOutSlowInEasing
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.core.spring
import androidx.compose.animation.core.tween
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.combinedClickable
import androidx.compose.foundation.interaction.MutableInteractionSource
import androidx.compose.foundation.interaction.collectIsPressedAsState
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxScope
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.Alignment
import androidx.compose.ui.draw.clip
import androidx.compose.ui.draw.drawWithContent
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.CompositingStrategy
import androidx.compose.ui.graphics.Outline
import androidx.compose.ui.graphics.Shape
import androidx.compose.ui.graphics.TransformOrigin
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.graphics.luminance
import androidx.compose.ui.input.pointer.PointerEventPass
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.launch
import kotlin.math.abs

/** Clean white canvas for the glass treatment. Colour comes from subtle blue light, not a gradient page. */
@Composable
fun GlassBackdrop(modifier: Modifier = Modifier, content: @Composable BoxScope.() -> Unit) {
    Box(modifier = modifier.background(androidx.compose.material3.MaterialTheme.colorScheme.background)) { content() }
}

/**
 * A restrained glass surface for a white canvas: translucent white, a cool blue inner sheen and a
 * bright edge. The shadow is intentionally soft enough to read as light passing through glass.
 */
@Composable
fun Modifier.glass(
    shape: Shape = RoundedCornerShape(28.dp),
    tint: Color = Color.White.copy(alpha = .88f),
    borderAlpha: Float = .82f,
    accent: Color? = null,
    shadowElevation: Dp = 14.dp,
): Modifier {
    val isDark = tint.luminance() < .5f
    val resolvedAccent = accent ?: MaterialTheme.colorScheme.primary
    val shadow = if (shadowElevation > 0.dp) {
        Modifier.shadow(
            elevation = shadowElevation,
            shape = shape,
            ambientColor = if (isDark) Color.Black.copy(alpha = .50f) else resolvedAccent.copy(alpha = .12f),
            spotColor = if (isDark) Color.Black.copy(alpha = .70f) else resolvedAccent.copy(alpha = .22f),
        )
    } else {
        Modifier
    }
    return this
    .then(shadow)
    .background(
        Brush.linearGradient(
            colors = if (isDark) {
                listOf(
                    tint.copy(alpha = .94f),
                    tint.copy(alpha = .88f),
                    tint.copy(alpha = .84f),
                )
            } else {
                listOf(
                    tint.copy(alpha = .96f),
                    tint.copy(alpha = .88f),
                    tint.copy(alpha = .82f),
                )
            },
            start = Offset(0f, 0f),
            end = Offset(600f, 600f),
        ),
        shape,
    )
    .border(1.25.dp, if (isDark) Color.White.copy(alpha = .14f) else Color.White.copy(alpha = borderAlpha), shape)
}

/**
 * A controllable blue drop shadow for large circular surfaces. This deliberately avoids
 * RenderNode elevation because very large colored shadows can be rasterized with a faceted
 * outline on some Android devices. A radial field lives in a larger parent box, so the shadow
 * cannot be clipped by the circle itself.
 */
@Composable
fun FloatingCircle(
    size: androidx.compose.ui.unit.Dp,
    modifier: Modifier = Modifier,
    fill: Color,
    border: Color = Color.White.copy(alpha = .82f),
    shadowColor: Color = Color(0xFF1687D0),
    content: @Composable BoxScope.() -> Unit,
) {
    Box(
        modifier = modifier.size(size + 72.dp),
        contentAlignment = Alignment.Center,
    ) {
        androidx.compose.foundation.Canvas(Modifier.fillMaxSize()) {
            val cx = this.size.width / 2f
            val cy = this.size.height / 2f + 12.dp.toPx()
            val r = this.size.width * .48f
            drawCircle(
                brush = Brush.radialGradient(
                    colors = listOf(
                        shadowColor.copy(alpha = .30f),
                        shadowColor.copy(alpha = .14f),
                        Color.Transparent,
                    ),
                    center = Offset(cx, cy),
                    radius = r + 28.dp.toPx(),
                ),
                radius = r + 28.dp.toPx(),
                center = Offset(cx, cy),
            )
        }
        Box(
            modifier = Modifier
                .size(size)
                .background(fill, androidx.compose.foundation.shape.CircleShape)
                .border(1.25.dp, border, androidx.compose.foundation.shape.CircleShape),
            content = content,
        )
    }
}

/** A small theme-coloured drop shadow beneath floating controls. */
@Composable
fun Modifier.blueGlow(
    shape: androidx.compose.ui.graphics.Shape = RoundedCornerShape(32.dp),
    color: Color = androidx.compose.material3.MaterialTheme.colorScheme.primary,
): Modifier = this.shadow(
    elevation = 34.dp,
    shape = shape,
    clip = false,
    ambientColor = color.copy(alpha = .24f),
    spotColor = color.copy(alpha = .40f),
)

/** A subtle elastic press response for interactive UI. */
@Composable
fun Modifier.elasticClickable(
    enabled: Boolean = true,
    onClick: () -> Unit,
): Modifier {
    return this.luminousElasticClickable(
        enabled = enabled,
        shape = CircleShape,
        onClick = onClick,
    )
}

/**
 * An expressive, touch-reactive interaction modifier providing:
 * - Touch-originated radial ripple
 * - Finger-following luminous highlight
 * - One-sided directional elastic deformation
 * - Whole-component physical translation with spring release overshoot
 * - Hard platform/GPU offscreen layer clipping
 */
@Composable
fun Modifier.luminousElasticClickable(
    enabled: Boolean = true,
    shape: Shape = CircleShape,
    maxStretchPercent: Float = 0.092f,
    maxTranslateDp: Dp = 2.6.dp,
    luminousRadiusScale: Float = 1.0f,
    onClick: (() -> Unit)? = null,
    onLongClick: (() -> Unit)? = null,
): Modifier {
    val interactionSource = remember { MutableInteractionSource() }
    val isInteractionPressed by interactionSource.collectIsPressedAsState()
    val scope = rememberCoroutineScope()

    var rawTouchPosition by remember { mutableStateOf<Offset?>(null) }
    var lastTouchPosition by remember { mutableStateOf<Offset?>(null) }
    var lastRippleOrigin by remember { mutableStateOf<Offset?>(null) }
    val rippleProgress = remember { Animatable(1f) }

    var targetDx by remember { mutableFloatStateOf(0f) }
    var targetDy by remember { mutableFloatStateOf(0f) }

    // Active press state derived from either pointer touch or interaction source
    val isPressed = rawTouchPosition != null || isInteractionPressed

    val animatedDx by animateFloatAsState(
        targetValue = targetDx,
        animationSpec = if (isPressed) {
            spring(
                stiffness = 550f,
                dampingRatio = 0.82f,
            )
        } else {
            spring(
                stiffness = 350f,
                dampingRatio = 0.52f,
            )
        },
        label = "elasticDx",
    )

    val animatedDy by animateFloatAsState(
        targetValue = targetDy,
        animationSpec = if (isPressed) {
            spring(
                stiffness = 550f,
                dampingRatio = 0.82f,
            )
        } else {
            spring(
                stiffness = 350f,
                dampingRatio = 0.52f,
            )
        },
        label = "elasticDy",
    )

    val animatedScale by animateFloatAsState(
        targetValue = if (isPressed) 0.95f else 1.0f,
        animationSpec = if (isPressed) {
            tween(durationMillis = 90, easing = FastOutSlowInEasing)
        } else {
            spring(
                stiffness = 420f,
                dampingRatio = 0.62f,
            )
        },
        label = "luminousElasticScale",
    )

    val luminousAlpha by animateFloatAsState(
        targetValue = if (isPressed) 1.0f else 0.0f,
        animationSpec = if (isPressed) {
            tween(durationMillis = 120, easing = FastOutSlowInEasing)
        } else {
            tween(durationMillis = 260, easing = FastOutSlowInEasing)
        },
        label = "luminousAlpha",
    )

    val surfaceLuminance = MaterialTheme.colorScheme.surface.luminance()
    val isDarkTheme = surfaceLuminance < 0.5f
    val primaryColor = MaterialTheme.colorScheme.primary

    // Vibrant, rich theme-adaptive luminous bloom colors:
    val centerColor = if (isDarkTheme) {
        Color.White.copy(alpha = 0.48f * luminousAlpha)
    } else {
        primaryColor.copy(alpha = 0.28f * luminousAlpha)
    }

    val outerColor = if (isDarkTheme) {
        Color(0xFF80D8FF).copy(alpha = 0.22f * luminousAlpha)
    } else {
        primaryColor.copy(alpha = 0.11f * luminousAlpha)
    }

    var mod = this.pointerInput(enabled) {
        if (!enabled) return@pointerInput
        awaitPointerEventScope {
            while (true) {
                val event = awaitPointerEvent(PointerEventPass.Initial)
                val change = event.changes.firstOrNull()
                if (change != null) {
                    if (change.pressed) {
                        val pos = change.position
                        val cx = size.width / 2f
                        val cy = size.height / 2f
                        if (rawTouchPosition == null) {
                            lastRippleOrigin = pos
                            scope.launch {
                                rippleProgress.snapTo(0f)
                                rippleProgress.animateTo(
                                    targetValue = 1f,
                                    animationSpec = tween(durationMillis = 420, easing = FastOutSlowInEasing),
                                )
                            }
                        }
                        rawTouchPosition = pos
                        lastTouchPosition = pos
                        if (cx > 0f && cy > 0f) {
                            targetDx = ((pos.x - cx) / cx).coerceIn(-1f, 1f)
                            targetDy = ((pos.y - cy) / cy).coerceIn(-1f, 1f)
                        }
                    } else {
                        rawTouchPosition = null
                        targetDx = 0f
                        targetDy = 0f
                    }
                }
            }
        }
    }

    mod = mod
        .graphicsLayer {
            val maxTranslatePx = maxTranslateDp.toPx()
            val stretchX = 1f + maxStretchPercent * abs(animatedDx) - 0.03f * abs(animatedDy)
            val stretchY = 1f + (maxStretchPercent * 1.06f) * abs(animatedDy) - 0.03f * abs(animatedDx)
            val netScaleX = animatedScale * stretchX
            val netScaleY = animatedScale * stretchY

            val pivotX = (0.5f - 0.30f * animatedDx).coerceIn(0.15f, 0.85f)
            val pivotY = (0.5f - 0.30f * animatedDy).coerceIn(0.15f, 0.85f)
            transformOrigin = TransformOrigin(pivotX, pivotY)

            scaleX = netScaleX
            scaleY = netScaleY
            translationX = animatedDx * maxTranslatePx
            translationY = animatedDy * maxTranslatePx
            clip = true
            this.shape = shape
            compositingStrategy = CompositingStrategy.Offscreen
        }
        .clip(shape)
        .drawWithContent {
            drawContent()

            val w = size.width
            val h = size.height
            val currentTouchCenter = rawTouchPosition ?: lastTouchPosition ?: Offset(w / 2f, h / 2f)

            if (luminousAlpha > 0.01f) {
                val baseBloomRadius = if (isDarkTheme) {
                    size.maxDimension * 0.75f
                } else {
                    size.minDimension * 1.15f + size.maxDimension * 0.22f
                }
                val bloomRadius = baseBloomRadius * luminousRadiusScale

                val luminousBrush = if (isDarkTheme) {
                    Brush.radialGradient(
                        colors = listOf(
                            centerColor,
                            outerColor,
                            Color.Transparent,
                        ),
                        center = currentTouchCenter,
                        radius = bloomRadius,
                    )
                } else {
                    Brush.radialGradient(
                        colorStops = arrayOf(
                            0.00f to centerColor,
                            0.38f to primaryColor.copy(alpha = 0.15f * luminousAlpha),
                            0.72f to outerColor.copy(alpha = 0.04f * luminousAlpha),
                            1.00f to Color.Transparent,
                        ),
                        center = currentTouchCenter,
                        radius = bloomRadius,
                    )
                }
                when (val outline = shape.createOutline(size, layoutDirection, this)) {
                    is Outline.Rounded -> drawRoundRect(
                        brush = luminousBrush,
                        cornerRadius = outline.roundRect.topLeftCornerRadius,
                    )
                    is Outline.Generic -> drawPath(
                        path = outline.path,
                        brush = luminousBrush,
                    )
                    is Outline.Rectangle -> drawRect(
                        brush = luminousBrush,
                    )
                }
            }

            val p = rippleProgress.value
            val rippleAlpha = (1f - p) * (if (isPressed) 1f else luminousAlpha)
            if (p < 1f && rippleAlpha > 0.01f && lastRippleOrigin != null) {
                val maxRippleRadius = size.maxDimension * 0.90f
                val minRippleRadius = 12.dp.toPx()
                val currentRippleRadius = minRippleRadius + (maxRippleRadius - minRippleRadius) * p
                val rippleOrigin = Offset(
                    lastRippleOrigin!!.x.coerceIn(0f, w),
                    lastRippleOrigin!!.y.coerceIn(0f, h),
                )

                val rippleBrush = Brush.radialGradient(
                    colors = listOf(
                        centerColor.copy(alpha = centerColor.alpha * 0.85f * rippleAlpha),
                        outerColor.copy(alpha = outerColor.alpha * 0.45f * rippleAlpha),
                        Color.Transparent,
                    ),
                    center = rippleOrigin,
                    radius = currentRippleRadius,
                )
                when (val outline = shape.createOutline(size, layoutDirection, this)) {
                    is Outline.Rounded -> drawRoundRect(
                        brush = rippleBrush,
                        cornerRadius = outline.roundRect.topLeftCornerRadius,
                    )
                    is Outline.Generic -> drawPath(
                        path = outline.path,
                        brush = rippleBrush,
                    )
                    is Outline.Rectangle -> drawRect(
                        brush = rippleBrush,
                    )
                }
            }
        }

    return if (onClick != null) {
        mod.combinedClickable(
            enabled = enabled,
            interactionSource = interactionSource,
            indication = null,
            onClick = onClick,
            onLongClick = onLongClick,
        )
    } else {
        mod
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/MechanicalText.kt`

```kotlin
package app.expressiveclock.ui

import android.animation.ValueAnimator
import androidx.compose.animation.*
import androidx.compose.animation.core.*
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.Canvas
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clipToBounds
import androidx.compose.ui.draw.blur
import androidx.compose.ui.draw.BlurredEdgeTreatment
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.drawscope.clipRect
import androidx.compose.ui.graphics.TransformOrigin
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.semantics.clearAndSetSemantics
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.buildAnnotatedString
import androidx.compose.ui.text.withStyle
import androidx.compose.ui.text.SpanStyle
import androidx.compose.ui.text.rememberTextMeasurer
import androidx.compose.ui.text.drawText
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.delay
import java.text.BreakIterator
import java.util.Locale
import kotlin.math.abs
import kotlin.math.min

private data class MechanicalGlyph(val text: String, val index: Int, val glyph: String)

/** Grapheme cells preserve spaces, accents and supplementary characters. */
internal fun mechanicalCells(text: String): List<String> {
    val iterator = BreakIterator.getCharacterInstance(Locale.ROOT).apply { setText(text) }
    return buildList {
        var start = iterator.first()
        var end = iterator.next()
        while (end != BreakIterator.DONE) {
            add(text.substring(start, end)); start = end; end = iterator.next()
        }
    }
}

internal fun mechanicalChanges(previous: String, next: String): List<Int> {
    val old = mechanicalCells(previous)
    val updated = mechanicalCells(next)
    return updated.indices.filter { updated[it] != old.getOrNull(it) }
}

internal fun mechanicalStagger(index: Int, selection: Boolean): Int =
    (index * if (selection) 22 else 18).coerceAtMost(if (selection) 180 else 72)

/** Fixed outer bounds and smoothly positioned cells prevent name-length changes from jumping. */
@Composable
internal fun MechanicalText(
    text: String, style: TextStyle, color: Color, modifier: Modifier = Modifier,
    digits: Boolean = false, selectionOnly: Boolean = false, selectionKey: Any = text, active: Boolean = true,
    progressiveBlur: Boolean = false,
    colorAt: (Int) -> Color = { color },
) {
    val density = LocalDensity.current
    val measurer = rememberTextMeasurer()
    val cells = remember(text) { mechanicalCells(text) }
    var previousText by remember { mutableStateOf(text) }
    var previousKey by remember { mutableStateOf(selectionKey) }
    var retainedCount by remember { mutableIntStateOf(cells.size) }
    val previousCells = remember(previousText) { mechanicalCells(previousText) }
    val selection = remember(text) { previousKey != selectionKey || !digits }
    val changed = remember(text) { mechanicalChanges(previousText, text) }
    SideEffect { previousText = text; previousKey = selectionKey; retainedCount = maxOf(retainedCount, cells.size) }
    val animate = active && ValueAnimator.areAnimatorsEnabled()
    val viewportHeight = with(density) { (style.lineHeight.takeIf { it.value.isFinite() } ?: style.fontSize * 1.3f).toDp() }
    BoxWithConstraints(modifier.height(viewportHeight)
        .clearAndSetSemantics { contentDescription = text }) {
        // Paragraph caret advances preserve kerning and actual word spaces. Measuring each
        // letter (or a space's ink box) separately can collapse spaces and crowd narrow glyphs.
        val rawWidth = measurer.measure(text, style).size.width.toFloat()
        val fit = minOf(1f, with(density) { maxWidth.toPx() } / rawWidth.coerceAtLeast(1f))
        val fitted = style.copy(fontSize = style.fontSize * fit, lineHeight = style.lineHeight * fit)
        if (!animate) {
            Text(buildAnnotatedString { cells.forEachIndexed { i, char -> withStyle(SpanStyle(color = colorAt(i))) { append(char) } } }, style = fitted, color = color, maxLines = 1,
                modifier = Modifier.align(Alignment.Center))
        } else {
            val digitWidth = (0..9).maxOf { measurer.measure(it.toString(), fitted).size.width }.toFloat()
            val paragraph = measurer.measure(text, fitted)
            var offset = 0
            val widths = cells.map { glyph ->
                val start = paragraph.getHorizontalPosition(offset, usePrimaryDirection = true)
                offset += glyph.length
                val end = paragraph.getHorizontalPosition(offset, usePrimaryDirection = true)
                if (digits && glyph.firstOrNull()?.isDigit() == true) digitWidth else abs(end - start)
            }
            val fullWidth = widths.sum()
            val measuredCellWidth = maxOf(digitWidth, (cells + previousCells + "W").maxOfOrNull {
                measurer.measure(it, fitted).size.width.toFloat()
            } ?: 1f)
            var widestCell by remember(fitted) { mutableFloatStateOf(measuredCellWidth) }
            val cellWidth = maxOf(widestCell, measuredCellWidth)
            SideEffect { widestCell = cellWidth }
            repeat(maxOf(retainedCount, cells.size)) { index ->
                val char = cells.getOrElse(index) { " " }
                // Remember this policy per target glyph: a normal seconds update must not
                // interrupt an unchanged hour/minute reel already moving to a selected city.
                val animateThisChange = remember(char) { !selectionOnly || selection }
                val before = widths.take(index).sum()
                val centre = (with(density) { maxWidth.toPx() } - fullWidth) / 2f + before +
                    widths.getOrElse(index) { 0f } / 2f
                val position by animateFloatAsState(centre - cellWidth / 2f,
                    tween(if (digits) 250 else 420, easing = FastOutSlowInEasing), label = "reelPosition")
                // New slots in a longer title must begin with the previous paragraph's
                // (empty) strip, not the target glyph, until their staggered turn arrives.
                var staged by remember {
                    mutableStateOf(MechanicalGlyph(previousText, index, previousCells.getOrElse(index) { "" }))
                }
                LaunchedEffect(text, char) {
                    val next = MechanicalGlyph(text, index, char)
                    if (next != staged) {
                        val rank = if (digits) changed.indexOf(index).coerceAtLeast(0) else index
                        if (animateThisChange) delay(mechanicalStagger(rank, selection).toLong())
                        staged = next
                    }
                }
                AnimatedContent(staged,
                    modifier = Modifier.width(with(density) { cellWidth.toDp() }).fillMaxHeight()
                        .graphicsLayer { translationX = position }
                        .then(if (progressiveBlur) Modifier.wrapContentHeight(unbounded = true)
                            .requiredHeight(viewportHeight + 16.dp)
                            .featheredVerticalViewport(active = true, fadeHeightFraction = .18f)
                        else Modifier.clipToBounds()),
                    contentAlignment = Alignment.Center,
                    transitionSpec = {
                        if (!animateThisChange) EnterTransition.None togetherWith ExitTransition.None
                        else ((slideInVertically(spring(dampingRatio = 1f, stiffness = 420f)) { it } + fadeIn(tween(100))) togetherWith
                            (slideOutVertically(tween(180, easing = FastOutSlowInEasing)) { -it } + fadeOut(tween(100))))
                            .using(SizeTransform(clip = !progressiveBlur))
                    }, label = "mechanicalCell") { reel ->
                    val tilt = if (animateThisChange) {
                        val angle by transition.animateFloat(transitionSpec = { tween(230) }, label = "reelTilt") { state ->
                            when (state) { EnterExitState.PreEnter -> -35f; EnterExitState.Visible -> 0f; EnterExitState.PostExit -> 35f }
                        }
                        angle
                    } else 0f
                    val blur = if (progressiveBlur && animateThisChange) {
                        val travel by transition.animateFloat(transitionSpec = {
                            if (targetState == EnterExitState.Visible) spring(dampingRatio = 1f, stiffness = 420f)
                            else tween(180, easing = FastOutSlowInEasing)
                        }, label = "nameEdgeBlur") { state ->
                            when (state) {
                                EnterExitState.PreEnter -> 1f
                                EnterExitState.Visible -> 0f
                                EnterExitState.PostExit -> -1f
                            }
                        }
                        val edge = (abs(travel) / .75f).coerceIn(0f, 1f)
                        edge * edge * (3f - 2f * edge) * 4f
                    } else 0f
                    val glyphModifier = Modifier.blur(blur.dp, edgeTreatment = BlurredEdgeTreatment.Unbounded).graphicsLayer {
                            rotationX = tilt; cameraDistance = size.height * 8f
                            transformOrigin = TransformOrigin.Center
                        }
                    if (digits) Text(reel.glyph, style = fitted, color = colorAt(index), maxLines = 1, modifier = glyphModifier)
                    else {
                        // Reassemble clipped strips of the shaped paragraph, rather than laying out
                        // isolated letters. Narrow glyphs now keep their true bearings and kerning.
                        val source = remember(reel.text, fitted) { measurer.measure(reel.text, fitted) }
                        val sourceCells = remember(reel.text) { mechanicalCells(reel.text) }
                        val startOffset = sourceCells.take(reel.index).sumOf { it.length }
                        val endOffset = startOffset + (sourceCells.getOrNull(reel.index)?.length ?: 0)
                        val startX = source.getHorizontalPosition(startOffset, true)
                        val endX = source.getHorizontalPosition(endOffset, true)
                        val advance = abs(endX - startX)
                        Canvas(glyphModifier.width(with(density) { cellWidth.toDp() }).height(viewportHeight)) {
                            if (advance > 0f) {
                                val left = (size.width - advance) / 2f
                                clipRect(left = left, right = left + advance) {
                                    drawText(source, color = colorAt(index), topLeft = Offset(
                                        left - min(startX, endX), (size.height - source.size.height) / 2f))
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/MechanicalViewport.kt`

```kotlin
package app.expressiveclock.ui

import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.drawWithContent
import androidx.compose.ui.graphics.BlendMode
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.CompositingStrategy
import androidx.compose.ui.graphics.graphicsLayer

internal fun Modifier.featheredVerticalViewport(
    active: Boolean,
    fadeHeightFraction: Float = 0.18f,
): Modifier = if (!active) this else this
    .graphicsLayer {
        compositingStrategy = CompositingStrategy.Offscreen
    }
    .drawWithContent {
        drawContent()
        val fadePx = size.height * fadeHeightFraction
        val topFadeFraction = (fadePx / size.height).coerceIn(0f, 0.45f)
        val bottomFadeFraction = 1f - topFadeFraction

        val maskBrush = Brush.verticalGradient(
            0.0f to Color.Transparent,
            topFadeFraction to Color.Black,
            bottomFadeFraction to Color.Black,
            1.0f to Color.Transparent,
        )

        drawRect(
            brush = maskBrush,
            blendMode = BlendMode.DstIn,
        )
    }
```

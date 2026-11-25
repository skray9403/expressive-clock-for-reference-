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

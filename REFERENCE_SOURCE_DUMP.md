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


---

## `app/src/main/java/app/expressiveclock/ui/StopwatchDockIcon.kt`

```kotlin
/*
 * Outline adapted from Material Icons Outlined.Timer.
 * Copyright 2025 The Android Open Source Project. Licensed under Apache License 2.0.
 * https://www.apache.org/licenses/LICENSE-2.0
 */
package app.expressiveclock.ui

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.Box
import androidx.compose.material3.Icon
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.SolidColor
import androidx.compose.ui.graphics.drawscope.withTransform
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.graphics.vector.path
import androidx.compose.ui.unit.dp

// Same 24-unit outline and hand geometry as the existing icon; only the hand moves.
private val StopwatchCase = ImageVector.Builder("StopwatchCase", 24.dp, 24.dp, 24f, 24f).apply {
    path(fill = SolidColor(Color.Black)) {
        moveTo(15f, 1f); horizontalLineTo(9f); verticalLineToRelative(2f)
        horizontalLineToRelative(6f); verticalLineTo(1f); close()
        moveTo(19.03f, 7.39f); lineToRelative(1.42f, -1.42f)
        curveToRelative(-.43f, -.51f, -.9f, -.99f, -1.41f, -1.41f)
        lineToRelative(-1.42f, 1.42f)
        curveTo(16.07f, 4.74f, 14.12f, 4f, 12f, 4f)
        curveToRelative(-4.97f, 0f, -9f, 4.03f, -9f, 9f)
        reflectiveCurveToRelative(4.02f, 9f, 9f, 9f)
        reflectiveCurveToRelative(9f, -4.03f, 9f, -9f)
        curveTo(21f, 10.88f, 20.26f, 8.93f, 19.03f, 7.39f); close()
        moveTo(12f, 20f)
        curveToRelative(-3.87f, 0f, -7f, -3.13f, -7f, -7f)
        reflectiveCurveToRelative(3.13f, -7f, 7f, -7f)
        reflectiveCurveToRelative(7f, 3.13f, 7f, 7f)
        reflectiveCurveToRelative(-3.13f, 7f, -7f, 7f); close()
    }
}.build()

@Composable
internal fun StopwatchDockIcon(phase: () -> Float, tint: Color, modifier: Modifier = Modifier) {
    Box(modifier) {
        Icon(StopwatchCase, contentDescription = null, tint = tint, modifier = Modifier.matchParentSize())
        Canvas(Modifier.matchParentSize()) {
            withTransform({
                scale(size.width / 24f, size.height / 24f, Offset.Zero)
                rotate(phase(), Offset(12f, 13f))
            }) {
                drawRect(tint, topLeft = Offset(11f, 8f), size = Size(2f, 6f))
            }
        }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/TimerReadoutGeometry.kt`

```kotlin
package app.expressiveclock.ui

import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

internal data class TimerReadoutGeometry(val digitWidth: Dp, val colonWidth: Dp, val spacing: Dp, val capHeight: Dp)

/** Shared with the Timer setup readout: 1 : .28 cells, .08 gaps, 1.32 cap ratio. */
internal fun timerReadoutGeometry(availableWidth: Dp, text: String): TimerReadoutGeometry {
    val units = text.count { it != ':' } + text.count { it == ':' } * .28f + (text.length - 1).coerceAtLeast(0) * .08f
    val unit = if (units > 0) availableWidth / units else availableWidth
    return TimerReadoutGeometry(unit, unit * .28f, unit * .08f, (unit.value * 1.32f).dp)
}
```


---

## `app/src/main/java/app/expressiveclock/ui/responsive/AppWindowSizeInfo.kt`

```kotlin
package app.expressiveclock.ui.responsive

import androidx.compose.runtime.Composable
import androidx.compose.runtime.staticCompositionLocalOf
import androidx.compose.ui.platform.LocalConfiguration
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

private val TWO_PANE_MIN_WIDTH = 600.dp
private val TWO_PANE_MIN_HEIGHT = 600.dp

/**
 * Layout classes for responsive switching in Expressive Clock.
 *
 * - [COMPACT]: Either dimension is below 600dp (including phones in landscape).
 * - [EXPANDED]: Width and height are both at least 600dp (unfolded/tablet two-pane layout).
 */
enum class AppWindowWidthClass {
    COMPACT,
    EXPANDED,
}

/**
 * Window size information wrapper.
 */
data class AppWindowSizeInfo(
    val widthClass: AppWindowWidthClass,
    val screenWidthDp: Dp,
    val screenHeightDp: Dp,
) {
    /** Short windows need scrolling even when there are no laps or other list items yet. */
    val isHeightCompact: Boolean
        get() = screenHeightDp < TWO_PANE_MIN_HEIGHT

    val isExpanded: Boolean
        get() = widthClass == AppWindowWidthClass.EXPANDED

    val isCompact: Boolean
        get() = widthClass == AppWindowWidthClass.COMPACT
}

/**
 * Calculates the current window size info from both available window dimensions.
 *
 * Re-evaluates dynamically when screen width changes (e.g. folding/unfolding, window resizing).
 */
@Composable
fun rememberAppWindowSizeInfo(): AppWindowSizeInfo {
    val configuration = LocalConfiguration.current
    val widthDp = configuration.screenWidthDp.dp
    val heightDp = configuration.screenHeightDp.dp

    val widthClass = if (widthDp >= TWO_PANE_MIN_WIDTH && heightDp >= TWO_PANE_MIN_HEIGHT) {
        AppWindowWidthClass.EXPANDED
    } else {
        AppWindowWidthClass.COMPACT
    }

    return AppWindowSizeInfo(
        widthClass = widthClass,
        screenWidthDp = widthDp,
        screenHeightDp = heightDp,
    )
}

/**
 * CompositionLocal providing [AppWindowSizeInfo] throughout the UI hierarchy.
 */
val LocalAppWindowSizeInfo = staticCompositionLocalOf {
    AppWindowSizeInfo(
        widthClass = AppWindowWidthClass.COMPACT,
        screenWidthDp = 360.dp,
        screenHeightDp = 800.dp,
    )
}
```


---

## `app/src/main/java/app/expressiveclock/ui/responsive/PersistentScrollEdges.kt`

```kotlin
package app.expressiveclock.ui.responsive

import androidx.compose.foundation.ScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.layout
import androidx.compose.ui.layout.onSizeChanged
import androidx.compose.ui.unit.dp
import app.expressiveclock.ui.screens.progressiveListEdgeMask

/** Fixed viewport falloffs, with end gutters so the first/last controls can rest in clear space.
 * Whether edges are needed depends on measured content, never the current scroll offset.
 * Fitting content keeps its original geometry and does not incur a blur layer.
 */
@Composable
internal fun Modifier.persistentScrollEdges(state: ScrollState): Modifier {
    var viewport by remember { mutableIntStateOf(0) }
    var overflowing by remember { mutableStateOf(false) }
    return onSizeChanged { viewport = it.height }
        .progressiveListEdgeMask(if (overflowing) 1f else 0f, if (overflowing) 1f else 0f)
        .verticalScroll(state)
        .layout { measurable, constraints ->
            val child = measurable.measure(constraints)
            val needsEdges = viewport > 0 && child.height > viewport
            if (overflowing != needsEdges) overflowing = needsEdges
            val gutter = if (needsEdges) 32.dp.roundToPx() else 0
            layout(child.width, child.height + gutter * 2) { child.placeRelative(0, gutter) }
        }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/responsive/ScrollPane.kt`

```kotlin
package app.expressiveclock.ui.responsive

import app.expressiveclock.ui.responsive.persistentScrollEdges
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import app.expressiveclock.ui.screens.progressiveListEdgeMask

/** Centre unchanged when content fits; scroll with soft edges when the window becomes too short. */
@Composable
internal fun ScrollPane(modifier: Modifier = Modifier, content: @Composable ColumnScope.() -> Unit) {
    val state = rememberScrollState()
    BoxWithConstraints(modifier.fillMaxSize()) {
        Column(Modifier.fillMaxWidth()
            .persistentScrollEdges(state).heightIn(min = maxHeight),
            verticalArrangement = Arrangement.Center, horizontalAlignment = Alignment.CenterHorizontally,
            content = content)
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/responsive/TwoPaneLayout.kt`

```kotlin
package app.expressiveclock.ui.responsive

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxScope
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxHeight
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.runtime.Composable
import androidx.compose.ui.draw.clipToBounds
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

/**
 * A flexible, configurable two-pane layout foundation for expanded window widths.
 *
 * Supports:
 * - Configurable left-pane and right-pane composables.
 * - Configurable left/right weight proportions (e.g. 1f to 1f for equal split, or 0.38f to 0.62f for Master-Detail).
 * - Configurable horizontal gap between panes.
 * - Proper handling of independent pane bounds and edge clipping.
 */
@Composable
fun TwoPaneLayout(
    leftPane: @Composable BoxScope.() -> Unit,
    rightPane: @Composable BoxScope.() -> Unit,
    modifier: Modifier = Modifier,
    leftWeight: Float = 1f,
    rightWeight: Float = 1f,
    paneSpacing: Dp = 24.dp,
    contentPaddingDp: Dp = 24.dp,
    clipLeftPane: Boolean = true,
) {
    Row(
        modifier = modifier
            .fillMaxSize()
            .padding(horizontal = contentPaddingDp),
        horizontalArrangement = Arrangement.spacedBy(paneSpacing),
        verticalAlignment = Alignment.CenterVertically,
    ) {
        Box(
            modifier = Modifier
                .weight(leftWeight)
                .fillMaxHeight()
                .then(if (clipLeftPane) Modifier.clipToBounds() else Modifier),
            contentAlignment = Alignment.Center,
            content = leftPane,
        )
        Box(
            modifier = Modifier
                .weight(rightWeight)
                .fillMaxHeight()
                .clipToBounds(),
            contentAlignment = Alignment.Center,
            content = rightPane,
        )
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/ClockHandMotion.kt`

```kotlin
package app.expressiveclock.ui.screens

import androidx.compose.animation.core.CubicBezierEasing
import java.time.ZoneId

/** One immutable motion snapshot: target zone and both hand corrections change atomically. */
internal data class ClockHandMotion(
    val zone: ZoneId,
    val startedAt: Long = 0L,
    val durationMillis: Long = 0L,
    val hourCorrection: Float = 0f,
    val minuteCorrection: Float = 0f,
) {
    fun anglesAt(millis: Long): Pair<Float, Float> {
        val target = clockAngles(millis, zone)
        val progress = if (durationMillis <= 0L) 1f else
            ((millis - startedAt).toFloat() / durationMillis).coerceIn(0f, 1f)
        val remaining = 1f - settling.transform(progress)
        return target.first + hourCorrection * remaining to target.second + minuteCorrection * remaining
    }

    fun retarget(next: ZoneId, millis: Long, duration: Long = 680L): ClockHandMotion {
        val displayed = anglesAt(millis)
        val target = clockAngles(millis, next)
        return ClockHandMotion(next, millis, duration,
            shortestClockDelta(displayed.first, target.first),
            shortestClockDelta(displayed.second, target.second))
    }

    private companion object {
        val settling = CubicBezierEasing(.2f, .85f, .25f, 1.04f)
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/DialInteraction.kt`

```kotlin
package app.expressiveclock.ui.screens

import android.os.Build
import android.view.HapticFeedbackConstants
import androidx.compose.animation.core.Animatable
import androidx.compose.animation.core.spring
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalView
import androidx.compose.ui.semantics.Role
import androidx.compose.ui.semantics.role
import androidx.compose.ui.semantics.stateDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import app.expressiveclock.ui.luminousElasticClickable

/** The Stopwatch’s state-change bounce and platform haptics, shared by both dials. */
@Composable
internal fun rememberDialRunningFeedback(running: Boolean?, identity: Any? = Unit): Animatable<Float, androidx.compose.animation.core.AnimationVector1D> {
    val scale = remember(identity) { Animatable(1f) }
    var previous by remember(identity) { mutableStateOf<Boolean?>(null) }
    val view = LocalView.current
    LaunchedEffect(identity, running) {
        val before = previous
        previous = running
        if (before != null && running != null && before != running) {
            view.performHapticFeedback(if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
                if (running) HapticFeedbackConstants.GESTURE_START else HapticFeedbackConstants.GESTURE_END
            } else HapticFeedbackConstants.KEYBOARD_TAP)
            scale.snapTo(.95f)
            scale.animateTo(1f, spring(dampingRatio = .58f, stiffness = 380f))
        } else if (running == null) scale.snapTo(1f)
    }
    return scale
}

@Composable
internal fun Modifier.timerDialTap(enabled: Boolean, running: Boolean, onToggle: () -> Unit): Modifier =
    if (!enabled) this else semantics {
        role = Role.Button
        stateDescription = if (running) "Running timer. Tap to pause" else "Paused timer. Tap to resume"
    }.luminousElasticClickable(
        shape = CircleShape,
        maxStretchPercent = .04f,
        maxTranslateDp = 2.dp,
        luminousRadiusScale = .58f,
        onClick = onToggle,
    )

/** Keep the original 410dp face wherever it fits with 16dp side clearance.
 * Clearance includes the inside border and small elastic overshoot; the existing diffuse
 * lighting may feather beyond it, as before. Do not reserve its entire transparent tail.
 */
internal fun stopwatchFaceDiameter(width: Dp): Dp =
    minOf(410.dp, (width - 32.dp).coerceAtLeast(1.dp))
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/StackedDigits.kt`

```kotlin
package app.expressiveclock.ui.screens

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.ui.draw.clip
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.compose.foundation.interaction.collectIsPressedAsState
import androidx.compose.material3.Icon
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.semantics.clearAndSetSemantics
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import app.expressiveclock.ui.theme.ClockFace
import app.expressiveclock.ui.blueGlow
import app.expressiveclock.ui.glass
import app.expressiveclock.ui.luminousElasticClickable
import app.expressiveclock.ui.theme.Numerals

/**
 * The concept's timer readout: **one pair of digits per line**, right-aligned, with the
 * separators drawn as two large dots rather than set as a colon.
 *
 * Three lines of two digits instead of one line of eight is what lets the numerals be this big.
 * On one line, `00:00:09` across a 412 dp screen caps out around 60 dp of cap height, while
 * stacked it reaches 77 dp. The concept is not being decorative here; the stack is the only way to
 * get a glanceable countdown onto a phone.
 *
 * Measured off the render: the three rows are in exact register, laid out as a **3-cell tabular
 * string** (one separator cell plus two digit cells, cell advance 67.2 dp), and that block is
 * **centred on the screen**, not right-aligned. Row pitch is 115 dp baseline to baseline, an ink
 * gap of 35.2 dp. The separator dots sit in the leading cell on the baseline, 18 dp across with
 * their centres 40.8 dp apart, and row one's pictogram is centred in that same cell.
 *
 * The readout has its own cut of the same family, `wdth` 95 / `wght` 490 with the slashed zero.
 * That cut was measured off the render rather than borrowed from the alarm grid, which is a much
 * narrower and much lighter face and looked nothing like it.
 */
@Composable
fun StackedDigits(
    parts: List<String>,
    capHeight: Dp,
    color: Color,
    modifier: Modifier = Modifier,
    leading: ImageVector? = null,
    label: String? = null,
) {
    Column(
        modifier = modifier
            .fillMaxWidth()
            .then(
                if (label != null) {
                    Modifier.clearAndSetSemantics { contentDescription = label }
                } else {
                    Modifier
                }
            ),
        horizontalAlignment = Alignment.CenterHorizontally,
        // 35.2 dp measured. `Numerals` boxes are ink-tight, so this is the true ink gap with no
        // ascent padding quietly padding it out.
        verticalArrangement = Arrangement.spacedBy(35.dp),
    ) {
        parts.forEachIndexed { i, part ->
            DigitRow(
                part = part,
                capHeight = capHeight,
                color = color,
                leading = leading.takeIf { i == 0 },
                separator = i > 0,
            )
        }
    }
}

/**
 * One line of the readout: a leading cell holding a pictogram, colon dots or nothing, followed by
 * the digit pair.
 *
 * Public because things have to line up with it. The stopwatch's Start button is exactly as wide as
 * a `: ss` row, and the only way to be *exactly* that is to measure this composable rather than to
 * add up a cell constant and a guess at the digit advance.
 */
@Composable
fun DigitRow(
    part: String,
    capHeight: Dp,
    color: Color,
    modifier: Modifier = Modifier,
    leading: ImageVector? = null,
    separator: Boolean = false,
) {
    Row(modifier, verticalAlignment = Alignment.CenterVertically) {
        // The separator's own tabular cell, the same advance as a digit cell.
        Box(Modifier.width(SEPARATOR_CELL_DP.dp), contentAlignment = Alignment.Center) {
            when {
                leading != null -> androidx.compose.material3.Icon(
                    leading,
                    contentDescription = null,
                    tint = color,
                    modifier = Modifier.size(46.dp),
                )
                separator -> ColonDots(color)
            }
        }
        Numerals(
            text = part,
            capHeight = capHeight,
            color = color,
            // The concept's Plank cut: 0.66 wide over tall, stroke 0.139 of cap, slashed zero. Not
            // the alarm grid's condensed face and not its light off-weight.
            width = ClockFace.TIMER_WIDTH,
            weight = ClockFace.TIMER_WEIGHT,
            slashedZero = true,
        )
    }
}

/** Measured: 18 dp dots with their centres 40.8 dp apart, so 22.8 dp of clear space between. */
private const val SEPARATOR_CELL_DP = 67.2f
private val DOT_DIAMETER = 18.dp

/**
 * How far a row's **visible** left edge sits inside its layout box.
 *
 * The separator gets a whole tabular cell of the same advance as a digit, and the colon's dots are
 * centred in it. So the box starts a good deal to the left of any ink, and anything that lines up
 * with the box rather than with the colon comes out looking off-centre even though the number is
 * right. Derived from the two constants above rather than measured off a screenshot, so it cannot
 * drift away from them.
 */
val ROW_INK_INSET: Dp = (SEPARATOR_CELL_DP.dp - DOT_DIAMETER) / 2

@Composable
private fun ColonDots(color: Color) {
    Column(verticalArrangement = Arrangement.spacedBy(22.8.dp)) {
        repeat(2) {
            Surface(color = color, shape = CircleShape, modifier = Modifier.size(DOT_DIAMETER)) {}
        }
    }
}

/**
 * The wide action pill the concept puts under its timer.
 *
 * Measured at **136 dp tall** and fully rounded: an M3 Expressive extra-large button. This is the
 * one control on the screen, so it gets the size that says so. M3E tactic 4, contain for emphasis,
 * in its most literal form. An earlier build had it at 78 dp, which read as an ordinary button.
 */
/** Measured off the render's Pause button. */
const val PILL_HEIGHT_DP = 136f

@Composable
fun WidePill(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    container: Color = MaterialTheme.colorScheme.primaryContainer.copy(alpha = 0.58f),
    content: Color = MaterialTheme.colorScheme.onPrimaryContainer,
    icon: ImageVector? = null,
    outlined: Boolean = false,
    height: Dp = PILL_HEIGHT_DP.dp,
    compact: Boolean = false,
    fillWidth: Boolean = true,
    contentPaddingH: Dp = if (compact) 10.dp else 24.dp,
    contentAlpha: Float = 1f,
    glow: Boolean = true,
) {
    Box(
        modifier = modifier
            .then(if (fillWidth) Modifier.fillMaxWidth() else Modifier)
            .height(height)
            .luminousElasticClickable(
                shape = CircleShape,
                maxStretchPercent = 0.09f,
                maxTranslateDp = 2.4.dp,
                onClick = onClick,
            )
            .clip(CircleShape)
            .glass(
                CircleShape,
                if (outlined) MaterialTheme.colorScheme.surfaceContainerLow.copy(alpha = .94f)
                else container.copy(alpha = .40f),
                .86f,
                shadowElevation = if (glow) 18.dp else 0.dp,
            )
            .padding(horizontal = contentPaddingH),
        contentAlignment = Alignment.Center,
    ) {
        val textStyle = if (compact) MaterialTheme.typography.titleMediumEmphasized else MaterialTheme.typography.headlineSmallEmphasized
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.fillMaxWidth().graphicsLayer { alpha = contentAlpha },
            horizontalArrangement = Arrangement.Center,
        ) {
            if (icon != null) {
                Icon(
                    icon,
                    contentDescription = null,
                    modifier = Modifier.size(if (compact) 22.dp else 30.dp),
                    tint = content,
                )
                if (text.isNotEmpty()) Spacer(Modifier.width(if (compact) 8.dp else 14.dp))
            }
            if (text.isNotEmpty()) {
                FittedPillLabel(text, textStyle.copy(color = content), Modifier.weight(1f, fill = false))
            }
        }
    }
}

/** Fit the measured glyph run, including letter spacing and Android's nonlinear font scaling. */
internal fun pillLabelScale(width: Int, height: Int, availableWidth: Int, availableHeight: Int): Float =
    minOf(1f, availableWidth.toFloat() / width.coerceAtLeast(1), availableHeight.toFloat() / height.coerceAtLeast(1))
        .coerceAtLeast(0f)

@Composable
private fun FittedPillLabel(text: String, style: androidx.compose.ui.text.TextStyle, modifier: Modifier) {
    androidx.compose.ui.layout.SubcomposeLayout(modifier) { constraints ->
        val label = subcompose("label") {
            Text(text, style = style, maxLines = 1, softWrap = false)
        }.single().measure(androidx.compose.ui.unit.Constraints())
        val fit = pillLabelScale(label.width, label.height, constraints.maxWidth, constraints.maxHeight)
        val width = (label.width * fit).toInt().coerceIn(constraints.minWidth, constraints.maxWidth)
        val height = (label.height * fit).toInt().coerceIn(constraints.minHeight, constraints.maxHeight)
        layout(width, height) {
            label.placeWithLayer(0, 0) {
                scaleX = fit; scaleY = fit
                transformOrigin = androidx.compose.ui.graphics.TransformOrigin(0f, 0f)
            }
        }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/WideClockPages.kt`

```kotlin
package app.expressiveclock.ui.screens

import android.animation.ValueAnimator
import androidx.compose.animation.*
import androidx.compose.animation.core.*
import androidx.compose.foundation.layout.*
import androidx.compose.ui.input.nestedscroll.nestedScroll
import androidx.compose.ui.input.nestedscroll.NestedScrollConnection
import androidx.compose.ui.input.nestedscroll.NestedScrollSource
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.unit.Velocity
import androidx.compose.foundation.gestures.draggable
import androidx.compose.foundation.gestures.rememberDraggableState
import androidx.compose.foundation.gestures.Orientation
import androidx.compose.foundation.clickable
import kotlinx.coroutines.launch
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.graphics.LayerOutsets
import androidx.compose.ui.graphics.luminance
import androidx.compose.ui.input.pointer.PointerEventPass
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.semantics.clearAndSetSemantics
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import app.expressiveclock.core.WorldCity
import app.expressiveclock.ui.WorldClockDigitalReadout
import app.expressiveclock.ui.MechanicalText
import app.expressiveclock.ui.rememberSystem24HourFormat
import app.expressiveclock.ui.rememberWallTicker
import app.expressiveclock.ui.sheets.prettify
import app.expressiveclock.ui.theme.WorldClockType
import java.time.ZoneId
import java.time.format.DateTimeFormatter
import java.util.Locale
import kotlin.math.roundToInt

/** Enough room for a comfortable clock pane and two real city-card columns. */
internal fun useWideClockPages(width: Float, height: Float, fontScale: Float): Boolean =
    width >= maxOf(720f, 2f * (132f * fontScale * 2f + 26f) + 56f) && height >= 520f

/** A retained two-pane clock destination, with only its active functional pane accepting input. */
@Composable
internal fun WideClockPages(
    worldSelected: Boolean,
    clockOnly: Boolean = false,
    panesOnly: Boolean = false,
    clockProgress: (() -> Float)? = null,
    cities: List<WorldCity>, selectedZoneId: String?,
    onSelectCity: (WorldCity) -> Unit, onShowLocalTime: () -> Unit,
    alarmContent: @Composable (Modifier) -> Unit,
    showAlarmEditor: Boolean,
    alarmEditor: @Composable () -> Unit,
    modifier: Modifier = Modifier,
) {
    val presentation = LocalWorldClockPresentation.current
    val toggleScope = rememberCoroutineScope()
    var toggleJob by remember { mutableStateOf<kotlinx.coroutines.Job?>(null) }
    val locale = androidx.compose.ui.platform.LocalConfiguration.current.locales[0]
    val now by rememberWallTicker()
    val local = prettify(ZoneId.systemDefault().id).copy(region = "Local Time")
    val selected = cities.firstOrNull { it.zone.id == selectedZoneId } ?: local
    val clockCity = if (worldSelected) selected else local
    val motion = remember { Animatable(if (worldSelected) 1f else 0f) }
    LaunchedEffect(worldSelected) {
        val target = if (worldSelected) 1f else 0f
        if (motion.value == target && !motion.isRunning) return@LaunchedEffect
        if (ValueAnimator.areAnimatorsEnabled())
            motion.animateTo(target, spring(dampingRatio = 1f, stiffness = 300f, visibilityThreshold = .001f))
        else motion.snapTo(target)
    }
    val scheme = MaterialTheme.colorScheme
    val density = LocalDensity.current
    val use24h = rememberSystem24HourFormat()
    val scrollConnection = remember(worldSelected, clockOnly) {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                if (!worldSelected || clockOnly || source != NestedScrollSource.UserInput) return Offset.Zero
                if (available.y > 0f && presentation.grid.canScrollBackward) return Offset.Zero
                toggleJob?.cancel()
                val old = presentation.collapse.floatValue
                val range = with(density) { 280.dp.toPx() }
                presentation.collapse.floatValue = (old - available.y / range).coerceIn(0f, 1f)
                return Offset(0f, (old - presentation.collapse.floatValue) * range)
            }
            override suspend fun onPreFling(available: Velocity): Velocity {
                if (!worldSelected || clockOnly || presentation.collapse.floatValue in listOf(0f, 1f)) return Velocity.Zero
                val start = presentation.collapse.floatValue
                androidx.compose.animation.core.animate(start, worldClockSnapTarget(start, -available.y / density.density),
                    animationSpec = tween(260)) { value, _ -> presentation.collapse.floatValue = value }
                return Velocity(0f, available.y)
            }
        }
    }
    BoxWithConstraints(modifier.nestedScroll(scrollConnection).fillMaxSize().then(if (clockOnly) Modifier else Modifier.worldClockAmbient(scheme.primary, scheme.surface.luminance() < .5f))) {
        val pane = (maxWidth - 24.dp) / 2
        val cityFadeBounds = remember(pane) { cityLightLayerOutsets((pane - 26.dp) / 2) }
        val alarmFadeBounds = remember { LayerOutsets(32.dp) }
        val dial = minOf(pane - 24.dp, maxHeight - 148.dp, 320.dp).coerceAtLeast(100.dp)
        val top = ((maxHeight - dial - 124.dp) / 2).coerceAtLeast(8.dp)
        val leftClock = (pane - dial) / 2
        val rightClock = pane + 24.dp + leftClock
        val editorVisible = showAlarmEditor && !worldSelected
        val clockAlpha = animateFloatAsState(if (editorVisible) 0f else 1f, tween(200), label = "clockEditorVisibility")

        // Pager destinations retain their own two-pane content. Do not leave an invisible
        // outgoing pane over the clock: its touch blocker would swallow page swipes.
        if (!clockOnly && !worldSelected) Box(Modifier.width(pane).fillMaxHeight().graphicsLayer {
            outsets = alarmFadeBounds
            alpha = (1f - motion.value * 1.65f).coerceIn(0f, 1f)
            translationX = -24.dp.toPx() * motion.value
        }.inactiveClockPane(worldSelected), contentAlignment = Alignment.TopCenter) {
            alarmContent(Modifier.width(minOf(pane - 16.dp, 520.dp)).fillMaxHeight())
        }
        if (!clockOnly && worldSelected) Box(Modifier.offset(x = pane + 24.dp).width(pane).fillMaxHeight()
            .graphicsLayer {
                outsets = cityFadeBounds
                // Each card and its halo share one staggered entrance below. A second
                // pane fade compounds opacity and makes faint lighting arrive late.
            }
            .inactiveClockPane(!worldSelected), contentAlignment = Alignment.Center) {
            SavedLocationGrid(cities = cities, localCity = local,
                onShowLocalTime = onShowLocalTime, selectedZoneId = selectedZoneId,
                nowUtcMillis = now, onSelectCity = onSelectCity,
                state = LocalWorldClockPresentation.current.grid,
                entranceVisible = worldSelected, centreShortContent = true, modifier = Modifier.fillMaxSize())
        }
        // No AnimatedContent or keys around the clock: the hands and second-hand frame source
        // survive every Alarm/World switch, including interrupted position animations.
        if (!panesOnly) {
            Box(Modifier.offset {
                IntOffset(with(density) { (rightClock + (leftClock - rightClock) * (clockProgress?.invoke() ?: motion.value)).roundToPx() },
                    with(density) { top.roundToPx() })
            }.size(dial).graphicsLayer { alpha = clockAlpha.value; outsets = LayerOutsets(dial * .4f); clip = false }
                .then(if (editorVisible) Modifier.clearAndSetSemantics {} else Modifier)
                .then(if (worldSelected) Modifier.draggable(
                    orientation = Orientation.Vertical,
                    state = rememberDraggableState { delta ->
                        toggleJob?.cancel()
                        presentation.collapse.floatValue = (presentation.collapse.floatValue - delta / with(density) { dial.toPx() }).coerceIn(0f, 1f)
                    },
                    onDragStopped = { velocity ->
                        val start = presentation.collapse.floatValue
                        toggleJob = toggleScope.launch {
                            androidx.compose.animation.core.animate(start, worldClockSnapTarget(start, -velocity / density.density),
                                animationSpec = tween(260)) { value, _ -> presentation.collapse.floatValue = value }
                        }
                    },
                ).clickable(interactionSource = remember { androidx.compose.foundation.interaction.MutableInteractionSource() }, indication = null, onClickLabel = "Switch clock style") {
                    toggleJob?.cancel()
                    toggleJob = toggleScope.launch {
                        val start = presentation.collapse.floatValue
                        androidx.compose.animation.core.animate(start, if (start < .5f) 1f else 0f, animationSpec = tween(260)) { value, _ -> presentation.collapse.floatValue = value }
                    }
                } else Modifier), contentAlignment = Alignment.Center) {
                WorldClockHeader(clockCity, now, dial,
                    presentation.collapse.floatValue * (clockProgress?.invoke() ?: motion.value), showDetails = false)
            }

            Box(Modifier.offset {
                IntOffset(with(density) { ((pane + 24.dp) * (1f - (clockProgress?.invoke() ?: motion.value))).roundToPx() },
                    with(density) { (top + worldClockDetailsTop(dial, presentation.collapse.floatValue * (clockProgress?.invoke() ?: motion.value), use24h)).roundToPx() })
            }.width(pane).graphicsLayer { alpha = clockAlpha.value }
                .then(if (editorVisible) Modifier.clearAndSetSemantics {} else Modifier)) {
                AnimatedContent(worldSelected, transitionSpec = {
                    (fadeIn(tween(250, delayMillis = 100)) + slideInVertically(tween(250)) { it / 10 }) togetherWith
                        fadeOut(tween(160))
                }, label = "sharedClockDetails") { world ->
                    Column(Modifier.fillMaxWidth(), horizontalAlignment = Alignment.CenterHorizontally,
                        verticalArrangement = Arrangement.spacedBy(10.dp)) {
                        if (world) {
                            MechanicalText(selected.city, WorldClockType.city(24.sp), scheme.onSurface,
                                modifier = Modifier.fillMaxWidth(), selectionKey = selected.zone.id, progressiveBlur = true)
                            val description = selected.timeAt(now).format(DateTimeFormatter.ofPattern("zzzz · EEE, d MMM", locale))
                            WorldClockSubtitle(description)
                        } else {
                            val localTime = local.timeAt(now)
                            val digits = localTime.format(DateTimeFormatter.ofPattern(if (use24h) "HH:mm:ss" else "hh:mm:ss"))
                            Row(Modifier.fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
                                WorldClockDigitalReadout(digits, local.zone.id,
                                    scheme.onSurface, scheme.primary, modifier = Modifier.weight(1f))
                                if (!use24h) Text(localTime.format(DateTimeFormatter.ofPattern("a", locale)),
                                    style = WorldClockType.secondary(14.sp), color = scheme.onSurface)
                            }
                            Text(local.timeAt(now).format(DateTimeFormatter.ofPattern("EEEE, d MMMM", locale)),
                                style = MaterialTheme.typography.bodySmall, color = scheme.onSurfaceVariant)
                        }
                    }
                }
            }
        }
        if (!clockOnly && showAlarmEditor) Box(Modifier.offset(x = pane + 24.dp).width(pane).fillMaxHeight()
            .graphicsLayer { alpha = if (editorVisible) 1f else 0f }
            .inactiveClockPane(!editorVisible)) { alarmEditor() }
    }
}

/** Invisible outgoing panes neither intercept touches nor expose duplicate accessibility content. */
private fun Modifier.inactiveClockPane(inactive: Boolean): Modifier = if (!inactive) this else
    clearAndSetSemantics {}.pointerInput(Unit) {
        awaitPointerEventScope {
            while (true) awaitPointerEvent(PointerEventPass.Initial).changes.forEach { it.consume() }
        }
    }
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/Winder.kt`

```kotlin
package app.expressiveclock.ui.screens

import androidx.compose.animation.core.animate
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectHorizontalDragGestures
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.rememberUpdatedState
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.CornerRadius
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.StrokeCap
import androidx.compose.ui.graphics.lerp
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.semantics.ProgressBarRangeInfo
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.progressBarRangeInfo
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import kotlin.math.PI
import kotlin.math.abs
import kotlin.math.cos
import kotlin.math.roundToInt
import kotlin.math.sin
import kotlinx.coroutines.launch

/**
 * A spring-wound timer's dial: the disc seen **edge-on**, wearing M3 Expressive's slider anatomy.
 *
 * ## The drum
 *
 * A stepped slider was the wrong shape for this. A pomodoro timer is a knurled wheel you turn, and
 * what sells a wheel on a flat screen is perspective: graduations sit at equal angles *around* a
 * cylinder and are then projected, `x = centre + R·sin θ`, which bunches them toward the rim the
 * way a real drum's do. Depth is `cos θ`, and it drives **width and brightness**: the graduation
 * under the handle is thickest and brightest, and the rest thin and dim until they cross the
 * silhouette at ±90°. Height stays constant across the whole face: shortening them with distance
 * as well was tried and read as a ruler with a curve drawn on it.
 *
 * ## Why it now looks like Material
 *
 * The first version was a bespoke drawing that happened to sit in a Material app. This one borrows
 * the expressive **slider's** anatomy, measured off the concept's own audio player. The drum body
 * *is* the track, in `surfaceContainerHighest`, at an 18 dp radius rather than a full capsule,
 * because a capsule reads as a pill and this is meant to read as a drum.
 *
 * There is no separate handle. The graduation passing under the centre *becomes* the marker and
 * hands the role back as it leaves, so nothing appears or disappears.
 *
 * There is deliberately no "wound so far" fill. It was tried and it read as a separate chip stuck
 * to the left end: a drum's position is shown by which graduation is under the handle, not by how
 * much of a track is coloured.
 *
 * ## Snap and smoothness
 *
 * The drag accumulates in a float and the drawing follows that float, not the rounded value. The
 * earlier version snapped, and the cause was its own re-sync: it reset `pos` from the incoming
 * `value` whenever the two drifted by more than three-quarters of a unit, which during a drag is
 * constantly. Every emitted step therefore yanked the drum back onto a whole number. Now an
 * incoming value this control just emitted is recognised as its own echo and ignored; only a
 * genuinely external change re-seats the drum.
 */
@Composable
fun Winder(
    value: Int,
    range: IntRange,
    onValueChange: (Int) -> Unit,
    modifier: Modifier = Modifier,
    height: Dp = 76.dp,
    /**
     * Angle between adjacent graduations. At ~4.6° roughly forty are visible across the face,
     * which is what makes it read as knurling rather than as a ruler.
     */
    stepRadians: Float = 0.080f,
    majorEvery: Int = 5,
) {
    val snapSpec = MaterialTheme.motionScheme.fastSpatialSpec<Float>()
    val scope = rememberCoroutineScope()
    val accent = MaterialTheme.colorScheme.primary
    val ridge = MaterialTheme.colorScheme.onSurfaceVariant
    val track = MaterialTheme.colorScheme.surfaceContainerHighest
    val onChange by rememberUpdatedState(onValueChange)

    var pos by remember { mutableFloatStateOf(value.toFloat()) }
    // What this control last reported. An incoming `value` equal to it is our own echo.
    var emitted by remember { mutableIntStateOf(value) }
    if (value != emitted && abs(pos - value) > 0.5f) {
        pos = value.toFloat()
        emitted = value
    }

    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .height(height)
            .semantics {
                contentDescription = "Winder"
                progressBarRangeInfo = ProgressBarRangeInfo(
                    current = value.toFloat(),
                    range = range.first.toFloat()..range.last.toFloat(),
                    steps = range.last - range.first,
                )
            }
            .pointerInput(range) {
                detectHorizontalDragGestures(
                    // Snap. Let go between two graduations and the drum settles onto the nearer
                    // one rather than resting off-detent, which is what a real wound dial does
                    // and what makes the value under the marker trustworthy.
                    //
                    // Animated rather than assigned, on the expressive *fast spatial* spring:
                    // under-damped, so it overshoots a hair and comes back, the way a sprung
                    // mechanism would. Jumping the value would land in the right place and feel
                    // like nothing happened.
                    onDragEnd = {
                        val target = pos.roundToInt()
                            .coerceIn(range.first, range.last)
                            .toFloat()
                        scope.launch {
                            animate(initialValue = pos, targetValue = target, animationSpec = snapSpec) { v, _ ->
                                pos = v
                            }
                        }
                        if (pos.roundToInt() != emitted) {
                            emitted = pos.roundToInt()
                            onChange(emitted)
                        }
                    },
                ) { change, dx ->
                    change.consume()
                    // Radius comes from the drawn width, so a pixel of drag turns the drum by
                    // dx/R radians and finger and graduations travel together.
                    //
                    // Negated: the drum is a cylinder seen edge-on, so dragging *left* rolls its
                    // near face away from you and brings higher numbers up under the marker. That
                    // is the same direction a physical wound timer turns. Following the finger
                    // instead means pulling left counts down, which reads backwards on a dial.
                    val delta = -(dx / (size.width / 2f)) / stepRadians
                    pos = (pos + delta).coerceIn(range.first.toFloat(), range.last.toFloat())
                    val rounded = pos.roundToInt()
                    if (rounded != emitted) {
                        emitted = rounded
                        onChange(rounded)
                    }
                }
            },
    ) {
        val cx = size.width / 2f
        val cy = size.height / 2f
        val r = size.width / 2f
        val trackH = size.height
        val corner = 18.dp.toPx()
        val ridgeLen = trackH * 0.68f

        // Squarer than a stock slider: a fully-rounded capsule reads as a pill, and this is a
        // drum. 18 dp keeps it in the M3 shape scale without pretending to be a capsule.
        drawRoundRect(
            color = track,
            topLeft = Offset(0f, cy - trackH / 2f),
            size = Size(size.width, trackH),
            cornerRadius = CornerRadius(corner),
        )
        val halfTurn = (PI / 2 / stepRadians).toInt()
        // How near a graduation has to come before it starts turning into the marker. Wider than
        // one step, so two neighbours always share the transition and the handoff is continuous.
        val selectRange = stepRadians * 1.6f

        for (i in -halfTurn..halfTurn) {
            val unit = pos.roundToInt() + i
            val theta = (unit - pos) * stepRadians
            val depth = cos(theta)
            if (depth <= 0.02f) continue

            val x = cx + r * sin(theta)
            val inRange = unit >= range.first && unit <= range.last
            val major = unit % majorEvery == 0 && inRange

            // Selection, smoothstepped on angular distance from the centre. There is no separate
            // handle: the graduation passing under the middle *becomes* the marker (growing,
            // thickening and taking the accent colour) and hands the role to its neighbour on the
            // way out. The previous version drew a fixed bar and skipped any graduation near it,
            // which is why they blinked out of existence as they approached.
            val t = (1f - (abs(theta) / selectRange)).coerceIn(0f, 1f)
            val sel = t * t * (3f - 2f * t)

            val baseAlpha = ((if (major) 0.34f else 0.14f) + 0.66f * depth * depth) *
                (if (inRange) 1f else 0.22f)
            val h = ridgeLen * (1f + 0.22f * sel)
            val w = (6.4f * (0.26f + 0.74f * depth)) * (1f + 1.05f * sel)
            val color = lerp(
                (if (major) accent else ridge).copy(alpha = baseAlpha.coerceIn(0f, 1f)),
                accent,
                sel,
            )

            drawLine(
                color = color,
                start = Offset(x, cy - h / 2f),
                end = Offset(x, cy + h / 2f),
                strokeWidth = w,
                cap = StrokeCap.Round,
            )
        }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/screens/WorldClockSnap.kt`

```kotlin
package app.expressiveclock.ui.screens

internal data class WorldClockHeaderVisibility(val analogue: Float, val digital: Float)

/** Pure scroll-derived handover: no independent alpha animations can outlive a reversal. */
internal fun worldClockHeaderVisibility(progress: Float): WorldClockHeaderVisibility {
    val collapse = progress.coerceIn(0f, 1f)
    return WorldClockHeaderVisibility(
        analogue = (1f - collapse / .45f).coerceIn(0f, 1f),
        digital = ((collapse - .55f) / .45f).coerceIn(0f, 1f),
    )
}

/** Velocity is in dp/s, positive toward collapse. Slow releases use the halfway threshold. */
internal fun worldClockSnapTarget(progress: Float, velocity: Float): Float = when {
    progress <= 0f -> 0f
    progress >= 1f -> 1f
    velocity >= 600f -> 1f
    velocity <= -600f -> 0f
    progress >= .5f -> 1f
    else -> 0f
}
```


---

## `app/src/main/java/app/expressiveclock/ui/sheets/PillDial.kt`

```kotlin
package app.expressiveclock.ui.sheets

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.TimePickerSelectionMode
import androidx.compose.material3.TimePickerState
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.StrokeCap
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.semantics.contentDescription
import androidx.compose.ui.semantics.semantics
import androidx.compose.ui.text.drawText
import androidx.compose.ui.text.rememberTextMeasurer
import androidx.compose.ui.unit.dp
import app.expressiveclock.ui.theme.ClockFace
import kotlin.math.PI
import kotlin.math.atan2
import kotlin.math.cos
import kotlin.math.max
import kotlin.math.sin
import kotlin.math.sqrt

/** A full circular alarm-time dial, intentionally matching the stopwatch face. */
@Composable
fun PillDial(
    state: TimePickerState,
    modifier: Modifier = Modifier,
    onHourPicked: () -> Unit = {},
) {
    val measurer = rememberTextMeasurer()
    val density = LocalDensity.current
    val minutes = state.selection == TimePickerSelectionMode.Minute
    val rings = if (!minutes && state.is24hour) 2 else 1
    val dialColor = MaterialTheme.colorScheme.surfaceContainerHighest.copy(alpha = .74f)
    val ink = MaterialTheme.colorScheme.onSurface
    val accent = MaterialTheme.colorScheme.primary
    val onAccent = MaterialTheme.colorScheme.onPrimary
    val labelStyle = remember(density) {
        ClockFace.numerals(capHeight = 15.dp, width = 105f, weight = 520, tabular = false)
    }
    val label = if (minutes) "minutes" else "hour"

    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .aspectRatio(1f)
            .clip(androidx.compose.foundation.shape.CircleShape)
            .background(MaterialTheme.colorScheme.surfaceContainerLow.copy(alpha = .93f), androidx.compose.foundation.shape.CircleShape)
            .border(1.25.dp, Color.White.copy(alpha = .86f), androidx.compose.foundation.shape.CircleShape)
            .semantics {
                contentDescription = "Pick the $label. Currently " +
                    if (minutes) "${state.minute} minutes" else "${state.hour} o'clock"
            }
            .pointerInput(minutes, rings, state.is24hour) {
                detectTapGestures { p -> applyCircular(state, p, size.width.toFloat(), size.height.toFloat(), rings, minutes, onHourPicked) }
            }
            .pointerInput(minutes, rings, state.is24hour) {
                detectDragGestures(
                    onDragEnd = { if (!minutes) onHourPicked() },
                ) { change, _ ->
                    applyCircular(state, change.position, size.width.toFloat(), size.height.toFloat(), rings, minutes, null)
                }
            },
    ) {
        val diameter = minOf(size.width, size.height)
        val center = Offset(size.width / 2f, size.height / 2f)
        val radius = diameter / 2f
        drawCircle(color = dialColor, radius = radius)

        val ringStep = radius * .19f
        val selectedRadius = if (rings == 2 && state.hour >= 12) radius * .57f else radius * .72f
        val selectedAngle = if (minutes) state.minute * PI.toFloat() / 30f else (state.hour % 12) * PI.toFloat() / 6f
        val tip = Offset(
            center.x + sin(selectedAngle) * selectedRadius,
            center.y - cos(selectedAngle) * selectedRadius,
        )
        drawLine(accent, center, tip, strokeWidth = 3.dp.toPx(), cap = StrokeCap.Round)
        drawCircle(accent, 4.dp.toPx(), center)
        drawCircle(accent, 16.dp.toPx(), tip)

        if (!minutes) {
            val outerR = radius * .72f
            for (i in 0 until 12) {
                val value = if (i == 0) 12 else i
                val angle = i * PI.toFloat() / 6f
                drawDialLabel(
                    value.toString(),
                    center,
                    outerR,
                    angle,
                    measurer,
                    labelStyle,
                    if (value == currentHourValue(state)) onAccent else ink,
                    value == currentHourValue(state),
                )
            }
            if (rings == 2) {
                val innerR = radius * .53f
                for (i in 0 until 12) {
                    val value = if (i == 0) 24 else i + 12
                    val angle = i * PI.toFloat() / 6f
                    drawDialLabel(
                        if (value == 24) "00" else value.toString(),
                        center,
                        innerR,
                        angle,
                        measurer,
                        labelStyle,
                        if (valueMatchesHour(state.hour, value)) onAccent else ink,
                        valueMatchesHour(state.hour, value),
                    )
                }
            }
        } else {
            val minuteValue = state.minute
            val outerR = radius * .72f
            for (i in 0 until 12) {
                val value = i * 5
                val angle = i * PI.toFloat() / 6f
                drawDialLabel(
                    "%02d".format(value),
                    center,
                    outerR,
                    angle,
                    measurer,
                    labelStyle,
                    if (value == minuteValue) onAccent else ink,
                    value == minuteValue,
                )
            }
        }
    }
}

private fun androidx.compose.ui.graphics.drawscope.DrawScope.drawDialLabel(
    text: String,
    center: Offset,
    radius: Float,
    angle: Float,
    measurer: androidx.compose.ui.text.TextMeasurer,
    style: androidx.compose.ui.text.TextStyle,
    color: androidx.compose.ui.graphics.Color,
    selected: Boolean,
) {
    val laid = measurer.measure(text, style)
    val x = center.x + sin(angle) * radius - laid.size.width / 2f
    val y = center.y - cos(angle) * radius - laid.size.height / 2f
    drawText(laid, color, Offset(x, y))
}

private fun currentHourValue(state: TimePickerState): Int = if (state.is24hour) state.hour else (state.hour % 12).let { if (it == 0) 12 else it }

private fun valueMatchesHour(hour: Int, value: Int): Boolean = when (value) {
    24 -> hour == 0
    else -> hour == value
}

private fun applyCircular(
    state: TimePickerState,
    at: Offset,
    w: Float,
    h: Float,
    rings: Int,
    minutes: Boolean,
    onHourPicked: (() -> Unit)?,
) {
    val centerX = w / 2f
    val centerY = h / 2f
    val dx = at.x - centerX
    val dy = at.y - centerY
    if (dx * dx + dy * dy < 1f) return

    var turns = atan2(dx, -dy) / (2f * PI.toFloat())
    if (turns < 0f) turns += 1f
    val index = ((turns * 12f) + .5f).toInt() % 12

    if (minutes) {
        state.minute = index * 5
        return
    }

    if (rings == 2 && sqrt(dx * dx + dy * dy) < (minOf(w, h) / 2f) * .64f) {
        state.hour = if (index == 0) 0 else index + 12
    } else {
        state.hour = if (index == 0) 12 else index
    }
    onHourPicked?.invoke()
}
```


---

## `app/src/main/java/app/expressiveclock/ui/theme/ClockFace.kt`

```kotlin
package app.expressiveclock.ui.theme

import androidx.compose.foundation.text.BasicText
import androidx.compose.runtime.Composable
import androidx.compose.runtime.remember
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.TransformOrigin
import androidx.compose.ui.graphics.graphicsLayer
import androidx.compose.ui.layout.Layout
import androidx.compose.ui.unit.Constraints
import androidx.compose.ui.text.AnnotatedString
import androidx.compose.ui.text.PlatformTextStyle
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.rememberTextMeasurer
import androidx.compose.ui.text.font.Font
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontVariation
import androidx.compose.ui.text.font.FontStyle
import androidx.compose.ui.text.font.FontSynthesis
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.platform.LocalFontFamilyResolver
import androidx.compose.ui.unit.Density
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.em
import androidx.compose.ui.unit.sp
import app.expressiveclock.R

/**
 * The clock's numerals: **Google Sans Flex, driven on its own axes.** No custom glyphs.
 *
 * ## What this replaces, and why the first attempt was wrong
 *
 * The M3 Expressive research-concept clock sets its alarm times in a face whose `0` is about one
 * fifth as wide as it is tall. An earlier build concluded no licensable font could do that and
 * hand-drew ten digits as vector paths. That was wrong twice over.
 *
 * First, Google Sans Flex went open-source in 2025 (`docs/m3/library/google-sans-flex-font.md`)
 * and has **six axes**: weight, width, optical size, slant, grade and roundedness. The reason the
 * earlier measurement missed `wdth` is that `tools/build_gsflex.py`'s predecessor asked
 * `fonts.googleapis.com` for `opsz,wght` only, and gstatic answers such a request with a *partial
 * instance*: the other four axes are flattened to their defaults and struck from `fvar`. The font
 * in the APK genuinely had two axes. The font Google ships has six.
 *
 * Measured on the correctly-built binary, the `0` glyph gives:
 *
 * | `wdth` | `wght` | ink w/h | matches |
 * |---|---|---|---|
 * | 25 | 250 | 0.234 | the concept's alarm tiles (~0.20) |
 * | 100 | 250 | 0.699 | the concept's Plank timer (~0.72) |
 *
 * So the two cuts the concept uses are one font at two width settings, and there was never
 * anything to draw.
 *
 * Second, the *off* state is not an outline. Magnifying a disarmed stem shows a single uniform
 * stroke with no inner contour: it is a **lighter weight of the same face**. The earlier "proof"
 * of outlining was a bad test: scanning across a `0` gives stroke-hole-stroke for any zero, because
 * a zero has a counter. State lives on the `wght` axis, which is what a variable font is for.
 */
object ClockFace {

    /** Ultra-condensed, for the alarm grid. */
    const val CONDENSED = 25f

    /** Normal width, for body text and labels. */
    const val NORMAL = 100f

    /**
     * The timer and stopwatch readout, measured off the concept's Plank screen: a `0` that is
     * 0.66 as wide as it is tall with a stroke 0.139 of the cap, plus a **slashed zero**, which
     * Google Sans Flex ships as the `zero` OpenType feature. Far heavier than the alarm grid's off
     * state, which is what an earlier build wrongly reused here.
     */
    const val TIMER_WIDTH = 95f
    const val TIMER_WEIGHT = 490

    /**
     * Armed. Measured, not picked: the armed stem measures 0.0639 of the cap height in the render,
     * and this binary hits it at `wght` 261 with `opsz` 144 / `wdth` 25 (Brent-solved, then
     * verified by two independent rasterisers). The value is opsz-dependent, so do not quote it
     * without opsz=144.
     */
    const val WEIGHT_ON = 260

    /**
     * Disarmed. The off stem measures 0.0306 of the cap, almost exactly half the armed weight,
     * which solves to `wght` 119. A light weight at display size is explicitly sanctioned by the
     * spec, so this is not a hack: it is the documented use of the axis.
     */
    const val WEIGHT_OFF = 120

    /**
     * Digit ink height as a fraction of the em, measured off the bundled binary
     * (1455/2000 at `wdth` 25). Converts a wanted cap height into a font size.
     */
    const val DIGIT_EM = 0.728f

    /**
     * Distance from a text box's top edge down to the top of a digit, in em.
     *
     * `hhea.ascent` is 0.966 em and a digit reaches 0.728 em above the baseline, so with
     * `includeFontPadding = false` the ink starts 0.238 em below the box top. [Numerals] offsets
     * by exactly this, which is what makes a numeral's layout box equal to its ink.
     */
    const val INK_TOP_EM = 0.966f - DIGIT_EM

    fun fontSizeFor(capHeight: Dp): Dp = capHeight / DIGIT_EM

    /**
     * Capital cap height as a fraction of the em: `OS/2.sCapHeight` = 1432/2000, corroborated by
     * the flat capitals T, M, W, F and H all measuring exactly 1432 units. ('S' is 1475, because
     * round letters overshoot, which is why a single sampled glyph is the wrong thing to
     * calibrate on.)
     *
     * This was 0.604 for a while, read off a screenshot of the meridiem whose segmentation was
     * wrong. Everything sized through here came out 19% too big, which is how the day letters
     * ended up 38.5 dp tall next to a 32 dp switch.
     */
    private const val CAPITAL_EM = 0.716f

    /** Distance from a text box's top down to a capital's top, in em. */
    const val CAPITAL_INK_TOP_EM = 0.966f - CAPITAL_EM

    fun capitalSizeFor(capHeight: Dp): Dp = capHeight / CAPITAL_EM

    private val cache = HashMap<Int, FontFamily>()

    /** Google Sans Flex pinned to one point in axis space. */
    fun family(
        opticalSize: Float,
        width: Float = NORMAL,
        weight: Int = 400,
        grade: Float = 0f,
        roundness: Float = 0f,
    ): FontFamily {
        val key = (opticalSize.toInt() * 31 + width.toInt()) * 31 * 31 +
            weight * 31 + grade.toInt() * 7 + roundness.toInt()
        return cache.getOrPut(key) {
            FontFamily(
                Font(
                    resId = R.font.google_sans_flex,
                    weight = FontWeight(weight.coerceIn(1, 1000)),
                    variationSettings = FontVariation.Settings(
                        FontVariation.Setting("opsz", opticalSize.coerceIn(6f, 144f)),
                        FontVariation.Setting("wdth", width.coerceIn(25f, 151f)),
                        FontVariation.Setting("wght", weight.coerceIn(1, 1000).toFloat()),
                        FontVariation.Setting("GRAD", grade.coerceIn(0f, 100f)),
                        FontVariation.Setting("ROND", roundness.coerceIn(0f, 100f)),
                    ),
                ),
            )
        }
    }

    /**
     * A numeral style whose box hugs the glyphs.
     *
     * `includeFontPadding = false` plus a trimmed line height is what makes the box predictable:
     * left as-is, Android adds the font's own ascent/descent padding and every measurement taken
     * off the concept renders lands in the wrong place by a different amount at each size.
     */
    /** The meridiem's style, exposed so a caller can measure it before deciding its tracking. */
    fun capitals(capHeight: Dp, width: Float = 75f, weight: Int = 500, tracking: Float = 0f): TextStyle {
        val size = capitalSizeFor(capHeight)
        return TextStyle(
            fontFamily = family(opticalSize = size.value, width = width, weight = weight),
            fontSize = size.value.sp,
            letterSpacing = tracking.em,
            platformStyle = PlatformTextStyle(includeFontPadding = false),
        )
    }

    const val ROUNDED = 100f

    fun numerals(
        capHeight: Dp,
        width: Float,
        weight: Int,
        tabular: Boolean = true,
        tracking: Float = 0f,
        slashedZero: Boolean = false,
        roundness: Float = 0f,
    ): TextStyle = TextStyle(
        // opsz tracks the type size, clamped at the font's 144 max. Every one of the 30
        // documented styles sets opsz equal to its point size; the Do is explicit.
        fontFamily = family(
            opticalSize = fontSizeFor(capHeight).value,
            width = width,
            weight = weight,
            roundness = roundness,
        ),
        fontSize = fontSizeFor(capHeight).value.sp,
        letterSpacing = tracking.em,
        /*
         * Tabular by default, and the alarm grid uses it too.
         *
         * An earlier version of this comment argued the opposite, claiming tabular was right for a
         * ticking stopwatch but wrong for a static alarm time, because the concept's tiles look
         * proportional and forcing tabular "pads every 1 out to a zero's width". Measured against
         * this binary at `wdth` 25, that last claim is simply not true: `one.tf` has 301 units of
         * ink against `zero.tf`'s 344, so a tabular 1 here is 88% as wide as a 0, not half. At this
         * width the tabular 1 carries a full flag and reads as a digit, not as a stick in a box.
         *
         * What proportional actually costs is measurable and large. Digit advances at `wdth` 25
         * run from 324 (`1`) to 449 (`8`), a 39% spread, so a four-digit time varies from 0.645
         * to 0.820 em. On a 174 dp tile that is a 45 dp swing in the block's width, which lands as
         * a 22.6 dp swing in the side margins from one tile to the next. Tabular makes every time
         * exactly one width, which is the only way a *grid* of times can line up with itself.
         */
        fontFeatureSettings = buildString {
            append(if (tabular) "tnum, lnum" else "pnum, lnum")
            // The concept's timer digits are slashed zeros. `zero` is a real feature on this
            // binary, so this is the font's own alternate rather than anything drawn.
            if (slashedZero) append(", zero")
        },
        // Leaving this on adds the font's own ascent/descent padding, and every measurement taken
        // off the concept renders then lands wrong by a different amount at each size.
        platformStyle = PlatformTextStyle(includeFontPadding = false),
    )

    /**
     * Optical tracking for the condensed cut, in em, applied to **every** element of a time:
     * hour, minutes and meridiem alike.
     *
     * One value on purpose. An earlier version measured the minutes and solved a per-element
     * tracking so the meridiem justified to their exact width, which is what the render does; it
     * also meant the letter fit changed depending on which digits were showing, and that reads as
     * sloppy rather than as precise. A single em value keeps the fit identical everywhere and
     * scales with each element's own size for free.
     *
     * The spec sets tracking to 0 across the scale, so this is an optical correction and not spec.
     * It is needed because this binary at `wdth` 25 advances about 0.30 of cap height per digit
     * where the render's face advances 0.22; the ink matches, the side bearings do not.
     */
    /**
     * The last nudge to the tracking, kept as its own term rather than folded into the number
     * below, because the alarm tile subtracts exactly twice it from the gap between the hour and
     * the minutes. Widening the digit gaps and narrowing the block gap by the same total keeps the
     * time the width it was; burying the delta would make that relationship impossible to see, and
     * the next person to adjust one would silently break the other.
     */
    const val TRACKING_DELTA = 0.004f
    const val CONDENSED_TRACKING = -0.014f + TRACKING_DELTA

}

/**
 * The horizontal ink bounds of [text] in [style], in pixels: the offset from the pen origin to the
 * first ink, and the width of the ink itself.
 *
 * Compose reports *advance* widths, which include each glyph's side bearings. At `wdth` 25 those
 * are not a rounding error. This face advances about 0.30 of the cap per digit where its ink is
 * 0.22 wide, so a numeral laid out at x = 0 has its first stroke some way inside its own box.
 * Anything that has to sit flush against an edge, or be fitted to an exact width, needs the ink.
 *
 * `Paint.getTextBounds` is the only API that gives it. The Typeface comes from Compose's own
 * resolver, so the variation settings baked into the family (the whole point of this file) are
 * the ones measured; constructing a Typeface by hand here would silently measure `wdth` 100.
 */
class InkBounds(val left: Float, val width: Float, val advance: Float) {
    /** Distance from the last ink to the end of the reserved box. */
    val right: Float get() = advance - left - width
}

@Composable
fun rememberInkBounds(text: String, style: TextStyle): InkBounds {
    val resolver = LocalFontFamilyResolver.current
    val density = LocalDensity.current
    return remember(text, style, density) { inkBounds(resolver, density, text, style) }
}

/**
 * The measuring half of [rememberInkBounds], callable outside composition.
 *
 * Exposed so a caller can *solve* against it, bisecting a variable-font axis until a string
 * measures the width it has to be. That needs many measurements per frame and so cannot be a
 * `@Composable` loop.
 */
fun inkBounds(
    resolver: FontFamily.Resolver,
    density: Density,
    text: String,
    style: TextStyle,
): InkBounds {
    return run {
        val typeface = resolver.resolve(
            fontFamily = style.fontFamily,
            fontWeight = style.fontWeight ?: FontWeight.Normal,
            fontStyle = FontStyle.Normal,
            fontSynthesis = FontSynthesis.All,
        ).value as android.graphics.Typeface
        val paint = android.graphics.Paint().apply {
            this.typeface = typeface
            textSize = with(density) { style.fontSize.toPx() }
            letterSpacing = style.letterSpacing.value
            // **Including the feature settings.** Leaving these off measured the font's default
            // figures while the caller rendered `tnum` ones, and at `wdth` 25 those are not close:
            // a tabular zero advances 472 units against a proportional 438. Every advance and every
            // bearing this returns was then ~7% adrift from what would actually be drawn, which is
            // invisible until something tries to fit ink to an edge with it.
            fontFeatureSettings = style.fontFeatureSettings
            isAntiAlias = true
        }
        val r = android.graphics.Rect()
        paint.getTextBounds(text, 0, text.length, r)
        // Left bearing, ink width, and the advance the layout will actually reserve. The third is
        // what lets a caller work out the *right* bearing, which is the other half of centring ink.
        InkBounds(r.left.toFloat(), r.width().toFloat(), paint.measureText(text))
    }
}

/**
 * A numeral whose layout box **is its ink**, with no ascent above and no descent below.
 *
 * Digits have no descender and their tops are all one height, so a box clipped to exactly the cap
 * height is honest. It is also what makes composition possible: two numerals at different sizes
 * top-align by simply sitting in a `Row`, with no magic offset, because neither carries invisible
 * padding that scales with its font size. That invisible padding is what "nothing is aligned
 * correctly" actually was.
 */
/**
 * Capitals in an ink-tight box, for the meridiem.
 *
 * It needs the same treatment as [Numerals] for the same reason, and here the reason is exact: the
 * render composes minutes cap + gap + AM cap to equal the hour's cap, i.e. the AM sits on the
 * hour's baseline. A text box carrying its own ascent and descent makes that sum impossible to
 * hit, and the whole tile's content then measures taller than the hour and floats upward.
 */
@Composable
fun CapText(
    text: String,
    capHeight: Dp,
    color: Color,
    modifier: Modifier = Modifier,
    width: Float = 75f,
    weight: Int = 500,
    tracking: Float = 0f,
) {
    val style = ClockFace.capitals(capHeight, width, weight, tracking).copy(color = color)
    InkTight(text, style, capHeight, modifier)
}

/**
 * Capitals set to an exact cap height and squeezed to an exact width.
 *
 * For the alarm tiles' day letters, which have to stand as tall as the 32 dp switch beside them
 * while seven of them share about 78 dp. That cannot be done on the width axis: even at the `wdth`
 * 25 floor, "SMTWTFS" at a 32 dp cap measures 120 dp. So the axis goes to its floor first and a
 * horizontal scale closes the rest. Unlike an outlined numeral, where a squeeze thins the
 * verticals and leaves the horizontals alone, these are small solid glyphs where it does not show.
 *
 * A transform does not change layout width, so the reported width is scaled here too; otherwise
 * the row would claim 120 dp and shove the switch off the tile.
 */
@Composable
fun StretchedCaps(
    text: String,
    capHeight: Dp,
    color: Color,
    scaleX: Float,
    modifier: Modifier = Modifier,
    width: Float = ClockFace.CONDENSED,
    weight: Int = 600,
) {
    val style = ClockFace.capitals(capHeight, width, weight).copy(color = color)
    StretchedCaps(AnnotatedString(text), style, capHeight, scaleX, modifier)
}

/**
 * The same squeeze, over a string whose runs carry their own styling.
 *
 * This is the one the day letters use. Setting seven letters as **one** text run rather than seven
 * boxed ones is what makes their spacing the font's own: the shaper applies each glyph's side
 * bearings and any kerning pair between them, which is precisely the thing that "is the kerning
 * between each day consistent?" is asking about. Seven separate boxes cannot kern across a box
 * boundary, and any per-letter cell width invented to stand in for that is a worse guess than the
 * one the type designer already made.
 *
 * [scaleX] is then derived from the measured run by squeezing the whole line to the block's width,
 * so tracking and letterform are squeezed by the same factor and the relative fit never changes.
 */
@Composable
fun StretchedCaps(
    text: AnnotatedString,
    style: TextStyle,
    capHeight: Dp,
    scaleX: Float,
    modifier: Modifier = Modifier,
) {
    val measurer = rememberTextMeasurer()
    val layout = remember(text, style) { measurer.measure(text, style) }
    Layout(
        content = {
            BasicText(
                text = text,
                style = style,
                maxLines = 1,
                softWrap = false,
                modifier = Modifier.graphicsLayer {
                    this.scaleX = scaleX
                    transformOrigin = TransformOrigin(0f, 0.5f)
                },
            )
        },
        modifier = modifier,
    ) { measurables, _ ->
        val placeable = measurables.first().measure(
            Constraints(maxWidth = Constraints.Infinity, maxHeight = Constraints.Infinity),
        )
        val inkTop = (layout.firstBaseline - capHeight.toPx()).toInt()
        // A transform does not change layout width, so the reported width is scaled here too;
        // otherwise the row claims its unsqueezed width and shoves the switch off the tile.
        layout((placeable.width * scaleX).toInt(), capHeight.roundToPx()) {
            placeable.place(0, -inkTop)
        }
    }
}

@Composable
fun Numerals(
    text: String,
    capHeight: Dp,
    color: Color,
    modifier: Modifier = Modifier,
    width: Float = ClockFace.NORMAL,
    weight: Int = ClockFace.WEIGHT_ON,
    tabular: Boolean = true,
    tracking: Float = 0f,
    slashedZero: Boolean = false,
    roundness: Float = 0f,
) {
    val style = ClockFace.numerals(capHeight, width, weight, tabular, tracking, slashedZero, roundness)
        .copy(color = color)
    InkTight(text, style, capHeight, modifier)
}

/**
 * Lays text out so its **layout box is exactly the ink**: cap height, nothing above, nothing below.
 *
 * The offset comes from [TextLayoutResult.firstBaseline], measured, rather than from an assumed
 * ascent. Deriving it from `hhea.ascent` looked right and was not: it put the alarm tiles' day
 * letters about 5 dp below the switch they were supposed to line up with, because whatever
 * `includeFontPadding = false` actually uses for the first line's top is not that number. A
 * measured baseline needs no assumption at all, since ink top is simply `baseline − capHeight`.
 */
@Composable
private fun InkTight(text: String, style: TextStyle, capHeight: Dp, modifier: Modifier) {
    val measurer = rememberTextMeasurer()
    val layout = remember(text, style) { measurer.measure(text, style) }
    Layout(
        content = { BasicText(text = text, style = style, maxLines = 1, softWrap = false) },
        modifier = modifier,
    ) { measurables, _ ->
        // Unbounded: a text box is taller than its ink, and handing it the reported height would
        // squeeze the glyphs into half the space they need.
        val placeable = measurables.first().measure(
            Constraints(maxWidth = Constraints.Infinity, maxHeight = Constraints.Infinity),
        )
        val inkTop = (layout.firstBaseline - capHeight.toPx()).toInt()
        layout(placeable.width, capHeight.roundToPx()) { placeable.place(0, -inkTop) }
    }
}
```


---

## `app/src/main/java/app/expressiveclock/ui/theme/Palettes.kt`

```kotlin
package app.expressiveclock.ui.theme

import androidx.compose.material3.ColorScheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.ui.graphics.Color

/**
 * GENERATED. Do not hand-edit. Regenerate with `tools/gen_palettes.py`.
 *
 * The app's selectable palettes are intentionally distinct. Concept uses a teal-forward scheme
 * to match the concept's visual language, while the remaining palettes provide clearly different
 * accent families for users who want to customise the app.
 *
 */
enum class Palette(val displayName: String, val light: ColorScheme, val dark: ColorScheme) {

    CONCEPT(
        "Concept",
        lightColorScheme(
            primary = Color(0xFF006874),
            onPrimary = Color(0xFFFFFFFF),
            primaryContainer = Color(0xFF97F0FF),
            onPrimaryContainer = Color(0xFF001F24),
            inversePrimary = Color(0xFF4FD8EB),
            secondary = Color(0xFF4A6267),
            onSecondary = Color(0xFFFFFFFF),
            secondaryContainer = Color(0xFFCDE7EC),
            onSecondaryContainer = Color(0xFF051F24),
            tertiary = Color(0xFF4F635D),
            onTertiary = Color(0xFFFFFFFF),
            tertiaryContainer = Color(0xFFD1E8E1),
            onTertiaryContainer = Color(0xFF0C1F1B),
            background = Color(0xFFF5FAFB),
            onBackground = Color(0xFF171D1E),
            surface = Color(0xFFF5FAFB),
            onSurface = Color(0xFF171D1E),
            surfaceVariant = Color(0xFFDBE4E6),
            onSurfaceVariant = Color(0xFF3F484A),
            surfaceTint = Color(0xFF006874),
            inverseSurface = Color(0xFF2C3133),
            inverseOnSurface = Color(0xFFECF2F3),
            error = Color(0xFFBA1A1A),
            onError = Color(0xFFFFFFFF),
            errorContainer = Color(0xFFFFDAD6),
            onErrorContainer = Color(0xFF410002),
            outline = Color(0xFF6F797A),
            outlineVariant = Color(0xFFBFC8CA),
            scrim = Color(0xFF000000),
            surfaceBright = Color(0xFFF5FAFB),
            surfaceDim = Color(0xFFD5DBDC),
            surfaceContainer = Color(0xFFE9EFF0),
            surfaceContainerHigh = Color(0xFFE3E9EA),
            surfaceContainerHighest = Color(0xFFDDE4E5),
            surfaceContainerLow = Color(0xFFEFF4F5),
            surfaceContainerLowest = Color(0xFFFFFFFF),
            primaryFixed = Color(0xFF97F0FF),
            primaryFixedDim = Color(0xFF4FD8EB),
            onPrimaryFixed = Color(0xFF001F24),
            onPrimaryFixedVariant = Color(0xFF004F58),
            secondaryFixed = Color(0xFFCDE7EC),
            secondaryFixedDim = Color(0xFFB1CBD0),
            onSecondaryFixed = Color(0xFF051F24),
            onSecondaryFixedVariant = Color(0xFF334B50),
            tertiaryFixed = Color(0xFFD1E8E1),
            tertiaryFixedDim = Color(0xFFB5CCC5),
            onTertiaryFixed = Color(0xFF0C1F1B),
            onTertiaryFixedVariant = Color(0xFF374A45),
        ),
        darkColorScheme(
            primary = Color(0xFF4FD8EB),
            onPrimary = Color(0xFF00363D),
            primaryContainer = Color(0xFF004F58),
            onPrimaryContainer = Color(0xFF97F0FF),
            inversePrimary = Color(0xFF006874),
            secondary = Color(0xFFB1CBD0),
            onSecondary = Color(0xFF1C3438),
            secondaryContainer = Color(0xFF334B50),
            onSecondaryContainer = Color(0xFFCDE7EC),
            tertiary = Color(0xFFB5CCC5),
            onTertiary = Color(0xFF21332F),
            tertiaryContainer = Color(0xFF374A45),
            onTertiaryContainer = Color(0xFFD1E8E1),
            background = Color(0xFF0E1415),
            onBackground = Color(0xFFE0E7E8),
            surface = Color(0xFF0E1415),
            onSurface = Color(0xFFE0E7E8),
            surfaceVariant = Color(0xFF3F484A),
            onSurfaceVariant = Color(0xFFBFC8CA),
            surfaceTint = Color(0xFF4FD8EB),
            inverseSurface = Color(0xFFE0E7E8),
            inverseOnSurface = Color(0xFF2C3133),
            error = Color(0xFFFFB4AB),
            onError = Color(0xFF690005),
            errorContainer = Color(0xFF93000A),
            onErrorContainer = Color(0xFFFFDAD6),
            outline = Color(0xFF899294),
            outlineVariant = Color(0xFF3F484A),
            scrim = Color(0xFF000000),
            surfaceBright = Color(0xFF343A3B),
            surfaceDim = Color(0xFF0E1415),
            surfaceContainer = Color(0xFF1A2021),
            surfaceContainerHigh = Color(0xFF242A2B),
            surfaceContainerHighest = Color(0xFF2F3536),
            surfaceContainerLow = Color(0xFF161C1D),
            surfaceContainerLowest = Color(0xFF090F10),
            primaryFixed = Color(0xFF97F0FF),
            primaryFixedDim = Color(0xFF4FD8EB),
            onPrimaryFixed = Color(0xFF001F24),
            onPrimaryFixedVariant = Color(0xFF004F58),
            secondaryFixed = Color(0xFFCDE7EC),
            secondaryFixedDim = Color(0xFFB1CBD0),
            onSecondaryFixed = Color(0xFF051F24),
            onSecondaryFixedVariant = Color(0xFF334B50),
            tertiaryFixed = Color(0xFFD1E8E1),
            tertiaryFixedDim = Color(0xFFB5CCC5),
            onTertiaryFixed = Color(0xFF0C1F1B),
            onTertiaryFixedVariant = Color(0xFF374A45),
        ),
    ),
    VIOLET(
        "Violet",
        lightColorScheme(
        primary = Color(0xFF6850A5),
        onPrimary = Color(0xFFFDF7FF),
        primaryContainer = Color(0xFFCAB6FF),
        onPrimaryContainer = Color(0xFF442C7F),
        inversePrimary = Color(0xFFBBA2FE),
        secondary = Color(0xFF4D6645),
        onSecondary = Color(0xFFEBFFE0),
        secondaryContainer = Color(0xFFD3F1C7),
        onSecondaryContainer = Color(0xFF435B3C),
        tertiary = Color(0xFF376B21),
        onTertiary = Color(0xFFECFFDC),
        tertiaryContainer = Color(0xFFC2FFA2),
        onTertiaryContainer = Color(0xFF31641B),
        background = Color(0xFFFFF7FF),
        onBackground = Color(0xFF3D2A51),
        surface = Color(0xFFFFF7FF),
        onSurface = Color(0xFF3D2A51),
        surfaceVariant = Color(0xFFEFDBFF),
        onSurfaceVariant = Color(0xFF6B5681),
        surfaceTint = Color(0xFF6850A5),
        inverseSurface = Color(0xFF130A1E),
        inverseOnSurface = Color(0xFFA697B2),
        error = Color(0xFFAC3149),
        onError = Color(0xFFFFF7F7),
        errorContainer = Color(0xFFF76A80),
        onErrorContainer = Color(0xFF68001F),
        outline = Color(0xFF88729E),
        outlineVariant = Color(0xFFC1A8D8),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFFF7FF),
        surfaceDim = Color(0xFFE9D1FF),
        surfaceContainer = Color(0xFFF7E9FF),
        surfaceContainerHigh = Color(0xFFF3E2FF),
        surfaceContainerHighest = Color(0xFFEFDBFF),
        surfaceContainerLow = Color(0xFFFBF0FF),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFCAB6FF),
        primaryFixedDim = Color(0xFFBEA5FF),
        onPrimaryFixed = Color(0xFF2F1369),
        onPrimaryFixedVariant = Color(0xFF4D3689),
        secondaryFixed = Color(0xFFD3F1C7),
        secondaryFixedDim = Color(0xFFC5E2BA),
        onSecondaryFixed = Color(0xFF31492B),
        onSecondaryFixedVariant = Color(0xFF4D6645),
        tertiaryFixed = Color(0xFFC2FFA2),
        tertiaryFixedDim = Color(0xFFB4F095),
        onTertiaryFixed = Color(0xFF1E5107),
        onTertiaryFixedVariant = Color(0xFF3B6F24),
        ),
        darkColorScheme(
        primary = Color(0xFFD4C3FF),
        onPrimary = Color(0xFF4A3A72),
        primaryContainer = Color(0xFFC7B4F5),
        onPrimaryContainer = Color(0xFF403168),
        inversePrimary = Color(0xFF65568F),
        secondary = Color(0xFFBBCBB2),
        onSecondary = Color(0xFF364431),
        secondaryContainer = Color(0xFF1C2918),
        onSecondaryContainer = Color(0xFF98A990),
        tertiary = Color(0xFFEDFFDF),
        onTertiary = Color(0xFF376B21),
        tertiaryContainer = Color(0xFFBFFCA0),
        onTertiaryContainer = Color(0xFF2F6219),
        background = Color(0xFF120B1A),
        onBackground = Color(0xFFF1DFFF),
        surface = Color(0xFF120B1A),
        onSurface = Color(0xFFF1DFFF),
        surfaceVariant = Color(0xFF2D203C),
        onSurfaceVariant = Color(0xFFB7A4C7),
        surfaceTint = Color(0xFFD4C3FF),
        inverseSurface = Color(0xFFFFF7FF),
        inverseOnSurface = Color(0xFF5B5163),
        error = Color(0xFFFD6F85),
        onError = Color(0xFF490013),
        errorContainer = Color(0xFF8A1632),
        onErrorContainer = Color(0xFFFF97A3),
        outline = Color(0xFF7F6F90),
        outlineVariant = Color(0xFF504260),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF342545),
        surfaceDim = Color(0xFF120B1A),
        surfaceContainer = Color(0xFF1F152A),
        surfaceContainerHigh = Color(0xFF261A33),
        surfaceContainerHighest = Color(0xFF2D203C),
        surfaceContainerLow = Color(0xFF181021),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFC9B7F8),
        primaryFixedDim = Color(0xFFBCAAEA),
        onPrimaryFixed = Color(0xFF2D1D54),
        onPrimaryFixedVariant = Color(0xFF4B3C74),
        secondaryFixed = Color(0xFFE5F6DB),
        secondaryFixedDim = Color(0xFFD7E8CD),
        onSecondaryFixed = Color(0xFF3D4C38),
        onSecondaryFixedVariant = Color(0xFF596953),
        tertiaryFixed = Color(0xFFC2FFA2),
        tertiaryFixedDim = Color(0xFFB4F095),
        onTertiaryFixed = Color(0xFF1E5107),
        onTertiaryFixedVariant = Color(0xFF3B6F24),
        ),
    ),
    CORAL(
        "Coral",
        lightColorScheme(
        primary = Color(0xFF9D414C),
        onPrimary = Color(0xFFFFF7F6),
        primaryContainer = Color(0xFFFFAAB0),
        onPrimaryContainer = Color(0xFF701F2B),
        inversePrimary = Color(0xFFFA8A94),
        secondary = Color(0xFF3A656F),
        onSecondary = Color(0xFFEEFBFF),
        secondaryContainer = Color(0xFFC6F3FE),
        onSecondaryContainer = Color(0xFF325D66),
        tertiary = Color(0xFF006974),
        onTertiary = Color(0xFFEBFCFF),
        tertiaryContainer = Color(0xFF67ECFF),
        onTertiaryContainer = Color(0xFF00555E),
        background = Color(0xFFFFF8F7),
        onBackground = Color(0xFF532420),
        surface = Color(0xFFFFF8F7),
        onSurface = Color(0xFF532420),
        surfaceVariant = Color(0xFFFFDAD6),
        onSurfaceVariant = Color(0xFF884F4A),
        surfaceTint = Color(0xFF9D414C),
        inverseSurface = Color(0xFF1E0706),
        inverseOnSurface = Color(0xFFBE938E),
        error = Color(0xFFAA371C),
        onError = Color(0xFFFFF7F6),
        errorContainer = Color(0xFFFA7150),
        onErrorContainer = Color(0xFF671200),
        outline = Color(0xFFA86A64),
        outlineVariant = Color(0xFFE7A099),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFFF8F7),
        surfaceDim = Color(0xFFFFCFCA),
        surfaceContainer = Color(0xFFFFE9E6),
        surfaceContainerHigh = Color(0xFFFFE2DE),
        surfaceContainerHighest = Color(0xFFFFDAD6),
        surfaceContainerLow = Color(0xFFFFF0EF),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFFFAAB0),
        primaryFixedDim = Color(0xFFFF959E),
        onPrimaryFixed = Color(0xFF530818),
        onPrimaryFixedVariant = Color(0xFF7B2833),
        secondaryFixed = Color(0xFFC6F3FE),
        secondaryFixedDim = Color(0xFFB8E4EF),
        onSecondaryFixed = Color(0xFF1D4A54),
        onSecondaryFixedVariant = Color(0xFF3C6771),
        tertiaryFixed = Color(0xFF67ECFF),
        tertiaryFixedDim = Color(0xFF56DEF0),
        onTertiaryFixed = Color(0xFF004148),
        onTertiaryFixedVariant = Color(0xFF005F6A),
        ),
        darkColorScheme(
        primary = Color(0xFFFFAEB4),
        onPrimary = Color(0xFF682A31),
        primaryContainer = Color(0xFFF59EA4),
        onPrimaryContainer = Color(0xFF5C2128),
        inversePrimary = Color(0xFF904A51),
        secondary = Color(0xFFB2CBD1),
        onSecondary = Color(0xFF2C4449),
        secondaryContainer = Color(0xFF11292E),
        onSecondaryContainer = Color(0xFF8FA8AE),
        tertiary = Color(0xFFAEF4FF),
        onTertiary = Color(0xFF005E68),
        tertiaryContainer = Color(0xFF67ECFF),
        onTertiaryContainer = Color(0xFF00555E),
        background = Color(0xFF1B0907),
        onBackground = Color(0xFFFFDEDA),
        surface = Color(0xFF1B0907),
        onSurface = Color(0xFFFFDEDA),
        surfaceVariant = Color(0xFF3E1C19),
        onSurfaceVariant = Color(0xFFD59E98),
        surfaceTint = Color(0xFFFFAEB4),
        inverseSurface = Color(0xFFFFF8F7),
        inverseOnSurface = Color(0xFF6A4E4B),
        error = Color(0xFFFE7453),
        onError = Color(0xFF450900),
        errorContainer = Color(0xFF881F05),
        onErrorContainer = Color(0xFFFF9B82),
        outline = Color(0xFF996964),
        outlineVariant = Color(0xFF663D39),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF48211D),
        surfaceDim = Color(0xFF1B0907),
        surfaceContainer = Color(0xFF2C1210),
        surfaceContainerHigh = Color(0xFF351714),
        surfaceContainerHighest = Color(0xFF3E1C19),
        surfaceContainerLow = Color(0xFF230D0B),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFFFAAB0),
        primaryFixedDim = Color(0xFFF29CA2),
        onPrimaryFixed = Color(0xFF4B141C),
        onPrimaryFixedVariant = Color(0xFF713138),
        secondaryFixed = Color(0xFFDBF5FC),
        secondaryFixedDim = Color(0xFFCDE7EE),
        onSecondaryFixed = Color(0xFF344C51),
        onSecondaryFixedVariant = Color(0xFF51686E),
        tertiaryFixed = Color(0xFF67ECFF),
        tertiaryFixedDim = Color(0xFF56DEF0),
        onTertiaryFixed = Color(0xFF004148),
        onTertiaryFixedVariant = Color(0xFF005F6A),
        ),
    ),
    EMBER(
        "Ember",
        lightColorScheme(
        primary = Color(0xFF9B4620),
        onPrimary = Color(0xFFFFF7F5),
        primaryContainer = Color(0xFFFFAD8D),
        onPrimaryContainer = Color(0xFF6D2501),
        inversePrimary = Color(0xFFF98F63),
        secondary = Color(0xFF446379),
        onSecondary = Color(0xFFF5F9FF),
        secondaryContainer = Color(0xFFC8E6FF),
        onSecondaryContainer = Color(0xFF36556B),
        tertiary = Color(0xFF00668C),
        onTertiary = Color(0xFFF4F9FF),
        tertiaryContainer = Color(0xFF60C6FB),
        onTertiaryContainer = Color(0xFF003D55),
        background = Color(0xFFFFF8F5),
        onBackground = Color(0xFF50280B),
        surface = Color(0xFFFFF8F5),
        onSurface = Color(0xFF50280B),
        surfaceVariant = Color(0xFFFFDBC8),
        onSurfaceVariant = Color(0xFF855333),
        surfaceTint = Color(0xFF9B4620),
        inverseSurface = Color(0xFF1C0901),
        inverseOnSurface = Color(0xFFBC957E),
        error = Color(0xFFAC3434),
        onError = Color(0xFFFFF7F6),
        errorContainer = Color(0xFFF56965),
        onErrorContainer = Color(0xFF65000B),
        outline = Color(0xFFA56E4C),
        outlineVariant = Color(0xFFE3A47E),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFFF8F5),
        surfaceDim = Color(0xFFFFD0B6),
        surfaceContainer = Color(0xFFFFEADF),
        surfaceContainerHigh = Color(0xFFFFE3D4),
        surfaceContainerHighest = Color(0xFFFFDBC8),
        surfaceContainerLow = Color(0xFFFFF1EA),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFFFAD8D),
        primaryFixedDim = Color(0xFFFF996F),
        onPrimaryFixed = Color(0xFF4B1600),
        onPrimaryFixedVariant = Color(0xFF792D08),
        secondaryFixed = Color(0xFFC8E6FF),
        secondaryFixedDim = Color(0xFFB8D8F2),
        onSecondaryFixed = Color(0xFF234357),
        onSecondaryFixedVariant = Color(0xFF405F75),
        tertiaryFixed = Color(0xFF60C6FB),
        tertiaryFixedDim = Color(0xFF50B8EC),
        onTertiaryFixed = Color(0xFF002637),
        onTertiaryFixedVariant = Color(0xFF004762),
        ),
        darkColorScheme(
        primary = Color(0xFFFFBDA4),
        onPrimary = Color(0xFF6E331A),
        primaryContainer = Color(0xFFFFA987),
        onPrimaryContainer = Color(0xFF622A11),
        inversePrimary = Color(0xFF8F4D32),
        secondary = Color(0xFFB7C9D9),
        onSecondary = Color(0xFF31424F),
        secondaryContainer = Color(0xFF162834),
        onSecondaryContainer = Color(0xFF94A6B5),
        tertiary = Color(0xFF85D2FF),
        onTertiary = Color(0xFF004763),
        tertiaryContainer = Color(0xFF60C6FB),
        onTertiaryContainer = Color(0xFF003D55),
        background = Color(0xFF1A0A02),
        onBackground = Color(0xFFFFDFCE),
        surface = Color(0xFF1A0A02),
        onSurface = Color(0xFFFFDFCE),
        surfaceVariant = Color(0xFF3C1E0A),
        onSurfaceVariant = Color(0xFFD2A084),
        surfaceTint = Color(0xFFFFBDA4),
        inverseSurface = Color(0xFFFFF8F5),
        inverseOnSurface = Color(0xFF695041),
        error = Color(0xFFFF716C),
        onError = Color(0xFF490006),
        errorContainer = Color(0xFF8A1A1E),
        onErrorContainer = Color(0xFFFF9993),
        outline = Color(0xFF976C52),
        outlineVariant = Color(0xFF643F28),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF45230C),
        surfaceDim = Color(0xFF1A0A02),
        surfaceContainer = Color(0xFF2A1405),
        surfaceContainerHigh = Color(0xFF331907),
        surfaceContainerHighest = Color(0xFF3C1E0A),
        surfaceContainerLow = Color(0xFF210E04),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFFFAD8D),
        primaryFixedDim = Color(0xFFF29F7E),
        onPrimaryFixed = Color(0xFF491802),
        onPrimaryFixedVariant = Color(0xFF6F341B),
        secondaryFixed = Color(0xFFDBEDFE),
        secondaryFixedDim = Color(0xFFCDDFEF),
        onSecondaryFixed = Color(0xFF364753),
        onSecondaryFixedVariant = Color(0xFF526371),
        tertiaryFixed = Color(0xFF60C6FB),
        tertiaryFixedDim = Color(0xFF50B8EC),
        onTertiaryFixed = Color(0xFF002637),
        onTertiaryFixedVariant = Color(0xFF004762),
        ),
    ),
    CITRON(
        "Citron",
        lightColorScheme(
        primary = Color(0xFF393F00),
        onPrimary = Color(0xFFBCC757),
        primaryContainer = Color(0xFFDFEB76),
        onPrimaryContainer = Color(0xFF4F5600),
        inversePrimary = Color(0xFFF0FC85),
        secondary = Color(0xFF596384),
        onSecondary = Color(0xFFFFFFFF),
        secondaryContainer = Color(0xFFDBE1FF),
        onSecondaryContainer = Color(0xFF475170),
        tertiary = Color(0xFF4C5FB0),
        onTertiary = Color(0xFFFFFFFF),
        tertiaryContainer = Color(0xFFA6B5FF),
        onTertiaryContainer = Color(0xFF1A2F7F),
        background = Color(0xFFFDFFD8),
        onBackground = Color(0xFF363B00),
        surface = Color(0xFFFDFFD8),
        onSurface = Color(0xFF363B00),
        surfaceVariant = Color(0xFFE6F08C),
        onSurfaceVariant = Color(0xFF616914),
        surfaceTint = Color(0xFF393F00),
        inverseSurface = Color(0xFF0D0F00),
        inverseOnSurface = Color(0xFF9DA07B),
        error = Color(0xFFB23D21),
        onError = Color(0xFFFFFFFF),
        errorContainer = Color(0xFFFA7150),
        onErrorContainer = Color(0xFF671200),
        outline = Color(0xFF7D862F),
        outlineVariant = Color(0xFFB6C062),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFDFFD8),
        surfaceDim = Color(0xFFDFEB75),
        surfaceContainer = Color(0xFFF3F9B4),
        surfaceContainerHigh = Color(0xFFECF4A0),
        surfaceContainerHighest = Color(0xFFE6F08C),
        surfaceContainerLow = Color(0xFFFAFEC6),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFDFEB76),
        primaryFixedDim = Color(0xFFD1DD6A),
        onPrimaryFixed = Color(0xFF3D4300),
        onPrimaryFixedVariant = Color(0xFF586100),
        secondaryFixed = Color(0xFFDBE1FF),
        secondaryFixedDim = Color(0xFFC9D3F9),
        onSecondaryFixed = Color(0xFF343E5D),
        onSecondaryFixedVariant = Color(0xFF515B7B),
        tertiaryFixed = Color(0xFFA6B5FF),
        tertiaryFixedDim = Color(0xFF94A6FD),
        onTertiaryFixed = Color(0xFF001761),
        onTertiaryFixedVariant = Color(0xFF243888),
        ),
        darkColorScheme(
        primary = Color(0xFFF8FFB6),
        onPrimary = Color(0xFF5C631E),
        primaryContainer = Color(0xFFE9F29B),
        onPrimaryContainer = Color(0xFF545B16),
        inversePrimary = Color(0xFF5C631E),
        secondary = Color(0xFFC1C6DD),
        onSecondary = Color(0xFF3A4052),
        secondaryContainer = Color(0xFF202537),
        onSecondaryContainer = Color(0xFF9EA3B9),
        tertiary = Color(0xFFB8C3FF),
        onTertiary = Color(0xFF253989),
        tertiaryContainer = Color(0xFFA6B5FF),
        onTertiaryContainer = Color(0xFF1A2F7F),
        background = Color(0xFF0E0F08),
        onBackground = Color(0xFFE6E8C0),
        surface = Color(0xFF0E0F08),
        onSurface = Color(0xFFE6E8C0),
        surfaceVariant = Color(0xFF252713),
        onSurfaceVariant = Color(0xFFABAE88),
        surfaceTint = Color(0xFFF8FFB6),
        inverseSurface = Color(0xFFFCFAEC),
        inverseOnSurface = Color(0xFF56564C),
        error = Color(0xFFFE7453),
        onError = Color(0xFF450900),
        errorContainer = Color(0xFF881F05),
        onErrorContainer = Color(0xFFFF9B82),
        outline = Color(0xFF757856),
        outlineVariant = Color(0xFF474A2C),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF2B2E15),
        surfaceDim = Color(0xFF0E0F08),
        surfaceContainer = Color(0xFF191A0E),
        surfaceContainerHigh = Color(0xFF1F2111),
        surfaceContainerHighest = Color(0xFF252713),
        surfaceContainerLow = Color(0xFF13140A),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFE1E994),
        primaryFixedDim = Color(0xFFD2DB87),
        onPrimaryFixed = Color(0xFF3D4300),
        onPrimaryFixedVariant = Color(0xFF59601A),
        secondaryFixed = Color(0xFFDDE1F9),
        secondaryFixedDim = Color(0xFFCFD3EB),
        onSecondaryFixed = Color(0xFF393F51),
        onSecondaryFixedVariant = Color(0xFF555B6F),
        tertiaryFixed = Color(0xFFA6B5FF),
        tertiaryFixedDim = Color(0xFF94A6FD),
        onTertiaryFixed = Color(0xFF001761),
        onTertiaryFixedVariant = Color(0xFF243888),
        ),
    ),
    FOREST(
        "Forest",
        lightColorScheme(
        primary = Color(0xFF006E3D),
        onPrimary = Color(0xFFE7FFEA),
        primaryContainer = Color(0xFF97F7B6),
        onPrimaryContainer = Color(0xFF005F34),
        inversePrimary = Color(0xFF9AFAB9),
        secondary = Color(0xFF7E5637),
        onSecondary = Color(0xFFFFF7F4),
        secondaryContainer = Color(0xFFFFDCC4),
        onSecondaryContainer = Color(0xFF6F492B),
        tertiary = Color(0xFF964A07),
        onTertiary = Color(0xFFFFF7F4),
        tertiaryContainer = Color(0xFFFFA363),
        onTertiaryContainer = Color(0xFF5A2900),
        background = Color(0xFFE8FFF0),
        onBackground = Color(0xFF003A28),
        surface = Color(0xFFE8FFF0),
        onSurface = Color(0xFF003A28),
        surfaceVariant = Color(0xFFB7EFD2),
        onSurfaceVariant = Color(0xFF356953),
        surfaceTint = Color(0xFF006E3D),
        inverseSurface = Color(0xFF00120A),
        inverseOnSurface = Color(0xFF87A394),
        error = Color(0xFFAC3434),
        onError = Color(0xFFFFF7F6),
        errorContainer = Color(0xFFF56965),
        onErrorContainer = Color(0xFF65000B),
        outline = Color(0xFF51856D),
        outlineVariant = Color(0xFF88BDA3),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFE8FFF0),
        surfaceDim = Color(0xFFAAE7C9),
        surfaceContainer = Color(0xFFCDF8E1),
        surfaceContainerHigh = Color(0xFFC2F3DA),
        surfaceContainerHighest = Color(0xFFB7EFD2),
        surfaceContainerLow = Color(0xFFD8FCE8),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFF97F7B6),
        primaryFixedDim = Color(0xFF89E8A9),
        onPrimaryFixed = Color(0xFF004A27),
        onPrimaryFixedVariant = Color(0xFF006A3B),
        secondaryFixed = Color(0xFFFFDCC4),
        secondaryFixedDim = Color(0xFFFFCAA3),
        onSecondaryFixed = Color(0xFF5A371A),
        onSecondaryFixedVariant = Color(0xFF7A5334),
        tertiaryFixed = Color(0xFFFFA363),
        tertiaryFixedDim = Color(0xFFF4944F),
        onTertiaryFixed = Color(0xFF381700),
        onTertiaryFixedVariant = Color(0xFF683000),
        ),
        darkColorScheme(
        primary = Color(0xFFE9FFEB),
        onPrimary = Color(0xFF2E6A45),
        primaryContainer = Color(0xFFB9FACA),
        onPrimaryContainer = Color(0xFF24623E),
        inversePrimary = Color(0xFF2E6A45),
        secondary = Color(0xFFE3C0A6),
        onSecondary = Color(0xFF533B28),
        secondaryContainer = Color(0xFF362110),
        onSecondaryContainer = Color(0xFFBF9D85),
        tertiary = Color(0xFFFFB687),
        onTertiary = Color(0xFF693000),
        tertiaryContainer = Color(0xFFFFA363),
        onTertiaryContainer = Color(0xFF5A2900),
        background = Color(0xFF02110A),
        onBackground = Color(0xFFC7EFD9),
        surface = Color(0xFF02110A),
        onSurface = Color(0xFFC7EFD9),
        surfaceVariant = Color(0xFF052C1E),
        onSurfaceVariant = Color(0xFF8DB4A0),
        surfaceTint = Color(0xFFE9FFEB),
        inverseSurface = Color(0xFFE9FFF1),
        inverseOnSurface = Color(0xFF46594F),
        error = Color(0xFFFF716C),
        onError = Color(0xFF490006),
        errorContainer = Color(0xFF8A1A1E),
        onErrorContainer = Color(0xFFFF9993),
        outline = Color(0xFF597D6B),
        outlineVariant = Color(0xFF2B4F3F),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF063323),
        surfaceDim = Color(0xFF02110A),
        surfaceContainer = Color(0xFF031E14),
        surfaceContainerHigh = Color(0xFF042519),
        surfaceContainerHighest = Color(0xFF052C1E),
        surfaceContainerLow = Color(0xFF02170E),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFB0F1C2),
        primaryFixedDim = Color(0xFFA3E3B5),
        onPrimaryFixed = Color(0xFF034A28),
        onPrimaryFixedVariant = Color(0xFF2A6742),
        secondaryFixed = Color(0xFFFFDCC4),
        secondaryFixedDim = Color(0xFFF2CDB4),
        onSecondaryFixed = Color(0xFF523A28),
        onSecondaryFixedVariant = Color(0xFF715642),
        tertiaryFixed = Color(0xFFFFA363),
        tertiaryFixedDim = Color(0xFFF4944F),
        onTertiaryFixed = Color(0xFF381700),
        onTertiaryFixedVariant = Color(0xFF683000),
        ),
    ),
    LAGOON(
        "Lagoon",
        lightColorScheme(
        primary = Color(0xFF006A6D),
        onPrimary = Color(0xFFE3FEFF),
        primaryContainer = Color(0xFF68F0F6),
        onPrimaryContainer = Color(0xFF00585A),
        inversePrimary = Color(0xFF6BF3F9),
        secondary = Color(0xFF695F30),
        onSecondary = Color(0xFFFFF8E9),
        secondaryContainer = Color(0xFFF6E9AD),
        onSecondaryContainer = Color(0xFF5E5527),
        tertiary = Color(0xFF725D00),
        onTertiary = Color(0xFFFFF8ED),
        tertiaryContainer = Color(0xFFFDDB5E),
        onTertiaryContainer = Color(0xFF5E4D00),
        background = Color(0xFFEDFCFF),
        onBackground = Color(0xFF003840),
        surface = Color(0xFFEDFCFF),
        onSurface = Color(0xFF003840),
        surfaceVariant = Color(0xFFB2ECF7),
        onSurfaceVariant = Color(0xFF2B6771),
        surfaceTint = Color(0xFF006A6D),
        inverseSurface = Color(0xFF001114),
        inverseOnSurface = Color(0xFF84A2A8),
        error = Color(0xFFAC3434),
        onError = Color(0xFFFFF7F6),
        errorContainer = Color(0xFFF56965),
        onErrorContainer = Color(0xFF65000B),
        outline = Color(0xFF49838D),
        outlineVariant = Color(0xFF81BBC6),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFEDFCFF),
        surfaceDim = Color(0xFFA2E5F1),
        surfaceContainer = Color(0xFFC9F6FE),
        surfaceContainerHigh = Color(0xFFBEF1FB),
        surfaceContainerHighest = Color(0xFFB2ECF7),
        surfaceContainerLow = Color(0xFFDBF9FF),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFF68F0F6),
        primaryFixedDim = Color(0xFF57E2E7),
        onPrimaryFixed = Color(0xFF004345),
        onPrimaryFixedVariant = Color(0xFF006265),
        secondaryFixed = Color(0xFFF6E9AD),
        secondaryFixedDim = Color(0xFFE8DAA0),
        onSecondaryFixed = Color(0xFF4B4317),
        onSecondaryFixedVariant = Color(0xFF695F30),
        tertiaryFixed = Color(0xFFFDDB5E),
        tertiaryFixedDim = Color(0xFFEECD51),
        onTertiaryFixed = Color(0xFF483B00),
        onTertiaryFixedVariant = Color(0xFF695600),
        ),
        darkColorScheme(
        primary = Color(0xFFBBFCFF),
        onPrimary = Color(0xFF006568),
        primaryContainer = Color(0xFF9CF1F4),
        onPrimaryContainer = Color(0xFF005B5E),
        inversePrimary = Color(0xFF006A6D),
        secondary = Color(0xFFD0C7A2),
        onSecondary = Color(0xFF464025),
        secondaryContainer = Color(0xFF2B260D),
        onSecondaryContainer = Color(0xFFADA481),
        tertiary = Color(0xFFFFEAA8),
        onTertiary = Color(0xFF685500),
        tertiaryContainer = Color(0xFFFDDB5E),
        onTertiaryContainer = Color(0xFF5E4D00),
        background = Color(0xFF001114),
        onBackground = Color(0xFFC3EDF5),
        surface = Color(0xFF001114),
        onSurface = Color(0xFFC3EDF5),
        surfaceVariant = Color(0xFF002B30),
        onSurfaceVariant = Color(0xFF8AB2B9),
        surfaceTint = Color(0xFFBBFCFF),
        inverseSurface = Color(0xFFEDFCFF),
        inverseOnSurface = Color(0xFF44585C),
        error = Color(0xFFFF716C),
        onError = Color(0xFF490006),
        errorContainer = Color(0xFF8A1A1E),
        onErrorContainer = Color(0xFFFF9993),
        outline = Color(0xFF547C83),
        outlineVariant = Color(0xFF254E54),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF003138),
        surfaceDim = Color(0xFF001114),
        surfaceContainer = Color(0xFF001D22),
        surfaceContainerHigh = Color(0xFF002429),
        surfaceContainerHighest = Color(0xFF002B30),
        surfaceContainerLow = Color(0xFF00161A),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFF97EBEE),
        primaryFixedDim = Color(0xFF88DDE0),
        onPrimaryFixed = Color(0xFF004345),
        onPrimaryFixedVariant = Color(0xFF006265),
        secondaryFixed = Color(0xFFFBF1CA),
        secondaryFixedDim = Color(0xFFEDE3BC),
        onSecondaryFixed = Color(0xFF4E482C),
        onSecondaryFixedVariant = Color(0xFF6B6546),
        tertiaryFixed = Color(0xFFFDDB5E),
        tertiaryFixedDim = Color(0xFFEECD51),
        onTertiaryFixed = Color(0xFF483B00),
        onTertiaryFixedVariant = Color(0xFF695600),
        ),
    ),
    AZURE(
        "Azure",
        lightColorScheme(
        primary = Color(0xFF3C5CAA),
        onPrimary = Color(0xFFF9F8FF),
        primaryContainer = Color(0xFFAABFFF),
        onPrimaryContainer = Color(0xFF103785),
        inversePrimary = Color(0xFF85A3F7),
        secondary = Color(0xFF3D675A),
        onSecondary = Color(0xFFE4FFF3),
        secondaryContainer = Color(0xFFCCFBE9),
        onSecondaryContainer = Color(0xFF386255),
        tertiary = Color(0xFF006D51),
        onTertiary = Color(0xFFE5FFF1),
        tertiaryContainer = Color(0xFF8AFED2),
        onTertiaryContainer = Color(0xFF006148),
        background = Color(0xFFFAF8FF),
        onBackground = Color(0xFF233156),
        surface = Color(0xFFFAF8FF),
        onSurface = Color(0xFF233156),
        surfaceVariant = Color(0xFFDAE2FF),
        onSurfaceVariant = Color(0xFF505E86),
        surfaceTint = Color(0xFF3C5CAA),
        inverseSurface = Color(0xFF060D20),
        inverseOnSurface = Color(0xFF959CB5),
        error = Color(0xFFAC3434),
        onError = Color(0xFFFFF7F6),
        errorContainer = Color(0xFFF56965),
        onErrorContainer = Color(0xFF65000B),
        outline = Color(0xFF6C7AA3),
        outlineVariant = Color(0xFFA3B1DE),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFAF8FF),
        surfaceDim = Color(0xFFCED9FF),
        surfaceContainer = Color(0xFFEAEDFF),
        surfaceContainerHigh = Color(0xFFE2E7FF),
        surfaceContainerHighest = Color(0xFFDAE2FF),
        surfaceContainerLow = Color(0xFFF2F3FF),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFAABFFF),
        primaryFixedDim = Color(0xFF96B1FF),
        onPrimaryFixed = Color(0xFF002362),
        onPrimaryFixedVariant = Color(0xFF1E418E),
        secondaryFixed = Color(0xFFCCFBE9),
        secondaryFixedDim = Color(0xFFBEECDB),
        onSecondaryFixed = Color(0xFF254F43),
        onSecondaryFixedVariant = Color(0xFF426C5F),
        tertiaryFixed = Color(0xFF8AFED2),
        tertiaryFixedDim = Color(0xFF7BEFC4),
        onTertiaryFixed = Color(0xFF004D38),
        onTertiaryFixedVariant = Color(0xFF006D51),
        ),
        darkColorScheme(
        primary = Color(0xFFBECEFF),
        onPrimary = Color(0xFF304477),
        primaryContainer = Color(0xFFACBFFB),
        onPrimaryContainer = Color(0xFF263A6D),
        inversePrimary = Color(0xFF4A5D92),
        secondary = Color(0xFFB2CCC2),
        onSecondary = Color(0xFF2D453D),
        secondaryContainer = Color(0xFF122A23),
        onSecondaryContainer = Color(0xFF90AA9F),
        tertiary = Color(0xFFDAFFEC),
        onTertiary = Color(0xFF006B4F),
        tertiaryContainer = Color(0xFF8AFED2),
        onTertiaryContainer = Color(0xFF006148),
        background = Color(0xFF090E1B),
        onBackground = Color(0xFFDEE4FF),
        surface = Color(0xFF090E1B),
        onSurface = Color(0xFFDEE4FF),
        surfaceVariant = Color(0xFF1B253F),
        onSurfaceVariant = Color(0xFFA1AACB),
        surfaceTint = Color(0xFFBECEFF),
        inverseSurface = Color(0xFFFAF8FF),
        inverseOnSurface = Color(0xFF505565),
        error = Color(0xFFFF716C),
        onError = Color(0xFF490006),
        errorContainer = Color(0xFF8A1A1E),
        onErrorContainer = Color(0xFFFF9993),
        outline = Color(0xFF6B7593),
        outlineVariant = Color(0xFF3E4763),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF202B49),
        surfaceDim = Color(0xFF090E1B),
        surfaceContainer = Color(0xFF12192D),
        surfaceContainerHigh = Color(0xFF161F36),
        surfaceContainerHighest = Color(0xFF1B253F),
        surfaceContainerLow = Color(0xFF0D1323),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFACBFFB),
        primaryFixedDim = Color(0xFF9EB2ED),
        onPrimaryFixed = Color(0xFF0D2557),
        onPrimaryFixedVariant = Color(0xFF2F4377),
        secondaryFixed = Color(0xFFDCF7EB),
        secondaryFixedDim = Color(0xFFCEE9DD),
        onSecondaryFixed = Color(0xFF354D45),
        onSecondaryFixedVariant = Color(0xFF516961),
        tertiaryFixed = Color(0xFF8AFED2),
        tertiaryFixedDim = Color(0xFF7BEFC4),
        onTertiaryFixed = Color(0xFF004D38),
        onTertiaryFixedVariant = Color(0xFF006D51),
        ),
    ),
    FUCHSIA(
        "Fuchsia",
        lightColorScheme(
        primary = Color(0xFF994164),
        onPrimary = Color(0xFFFFF7F7),
        primaryContainer = Color(0xFFFFA8C5),
        onPrimaryContainer = Color(0xFF6D1E41),
        inversePrimary = Color(0xFFFA90B6),
        secondary = Color(0xFF396667),
        onSecondary = Color(0xFFE0FFFF),
        secondaryContainer = Color(0xFFC9FAFA),
        onSecondaryContainer = Color(0xFF336162),
        tertiary = Color(0xFF006B68),
        onTertiary = Color(0xFFE1FFFC),
        tertiaryContainer = Color(0xFF69F1EC),
        onTertiaryContainer = Color(0xFF005856),
        background = Color(0xFFFFF8F8),
        onBackground = Color(0xFF502435),
        surface = Color(0xFFFFF8F8),
        onSurface = Color(0xFF502435),
        surfaceVariant = Color(0xFFFFD9E3),
        onSurfaceVariant = Color(0xFF844F62),
        surfaceTint = Color(0xFF994164),
        inverseSurface = Color(0xFF1D070F),
        inverseOnSurface = Color(0xFFB9939E),
        error = Color(0xFFAC3149),
        onError = Color(0xFFFFF7F7),
        errorContainer = Color(0xFFF76A80),
        onErrorContainer = Color(0xFF68001F),
        outline = Color(0xFFA36A7D),
        outlineVariant = Color(0xFFE0A0B5),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFFFFF8F8),
        surfaceDim = Color(0xFFFFCDDB),
        surfaceContainer = Color(0xFFFFE8ED),
        surfaceContainerHigh = Color(0xFFFFE1E8),
        surfaceContainerHighest = Color(0xFFFFD9E3),
        surfaceContainerLow = Color(0xFFFFF0F3),
        surfaceContainerLowest = Color(0xFFFFFFFF),
        primaryFixed = Color(0xFFFFA8C5),
        primaryFixedDim = Color(0xFFFD93B9),
        onPrimaryFixed = Color(0xFF52062C),
        onPrimaryFixedVariant = Color(0xFF79274A),
        secondaryFixed = Color(0xFFC9FAFA),
        secondaryFixedDim = Color(0xFFBBEBEC),
        onSecondaryFixed = Color(0xFF204F50),
        onSecondaryFixedVariant = Color(0xFF3E6C6D),
        tertiaryFixed = Color(0xFF69F1EC),
        tertiaryFixedDim = Color(0xFF58E3DE),
        onTertiaryFixed = Color(0xFF004442),
        onTertiaryFixedVariant = Color(0xFF006360),
        ),
        darkColorScheme(
        primary = Color(0xFFFFC1D4),
        onPrimary = Color(0xFF70334B),
        primaryContainer = Color(0xFFFEACC7),
        onPrimaryContainer = Color(0xFF652B42),
        inversePrimary = Color(0xFF8C4A62),
        secondary = Color(0xFFB0CCCC),
        onSecondary = Color(0xFF2B4445),
        secondaryContainer = Color(0xFF102A2A),
        onSecondaryContainer = Color(0xFF8EA9A9),
        tertiary = Color(0xFF80FFFA),
        onTertiary = Color(0xFF00615F),
        tertiaryContainer = Color(0xFF69F1EC),
        onTertiaryContainer = Color(0xFF005856),
        background = Color(0xFF1A090F),
        onBackground = Color(0xFFFFDDE6),
        surface = Color(0xFF1A090F),
        onSurface = Color(0xFFFFDDE6),
        surfaceVariant = Color(0xFF3C1C27),
        onSurfaceVariant = Color(0xFFCF9EAD),
        surfaceTint = Color(0xFFFFC1D4),
        inverseSurface = Color(0xFFFFF8F8),
        inverseOnSurface = Color(0xFF674E56),
        error = Color(0xFFFD6F85),
        onError = Color(0xFF490013),
        errorContainer = Color(0xFF8A1632),
        onErrorContainer = Color(0xFFFF97A3),
        outline = Color(0xFF956977),
        outlineVariant = Color(0xFF633D4A),
        scrim = Color(0xFF000000),
        surfaceBright = Color(0xFF45202E),
        surfaceDim = Color(0xFF1A090F),
        surfaceContainer = Color(0xFF2A121B),
        surfaceContainerHigh = Color(0xFF331721),
        surfaceContainerHighest = Color(0xFF3C1C27),
        surfaceContainerLow = Color(0xFF210D14),
        surfaceContainerLowest = Color(0xFF000000),
        primaryFixed = Color(0xFFFEACC7),
        primaryFixedDim = Color(0xFFEF9FBA),
        onPrimaryFixed = Color(0xFF4C162D),
        onPrimaryFixedVariant = Color(0xFF70334B),
        secondaryFixed = Color(0xFFDAF6F6),
        secondaryFixedDim = Color(0xFFCCE8E8),
        onSecondaryFixed = Color(0xFF334C4D),
        onSecondaryFixedVariant = Color(0xFF4F6969),
        tertiaryFixed = Color(0xFF69F1EC),
        tertiaryFixedDim = Color(0xFF58E3DE),
        onTertiaryFixed = Color(0xFF004442),
        onTertiaryFixedVariant = Color(0xFF006360),
        ),
    );
}

/** The two schemes the concept itself was drawn in, kept named for the places that reference them. */
internal val ConceptLight: ColorScheme = Palette.CONCEPT.light
internal val ConceptDark: ColorScheme = Palette.CONCEPT.dark
```


---

## `app/src/main/java/app/expressiveclock/ui/theme/Theme.kt`

```kotlin
package app.expressiveclock.ui.theme

import android.graphics.Color as AndroidColor
import android.os.Build
import androidx.activity.ComponentActivity
import androidx.activity.SystemBarStyle
import androidx.activity.compose.LocalActivity
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.ColorScheme
import androidx.compose.material3.LocalTonalElevationEnabled
import androidx.compose.material3.MaterialExpressiveTheme
import androidx.compose.material3.MotionScheme
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.CompositionLocalProvider
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.rememberUpdatedState
import android.view.ViewTreeObserver
import androidx.compose.runtime.remember
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.core.view.WindowCompat
import app.expressiveclock.data.DarkMode
import app.expressiveclock.data.ThemeSettings

/**
 * The theme root.
 *
 * [MaterialExpressiveTheme] rather than `MaterialTheme` is the single most consequential line here:
 * it installs the expressive motion scheme, shapes and the emphasized type slots. It carries no
 * opt-in annotation in 1.5.0-alpha25; the theming layer went stable ahead of the components.
 *
 * ## One scheme for the whole app
 *
 * An earlier build ran Alarms and World Clock dark while Timers and Stopwatch were light, because
 * that is how the concept renders happened to be published. As a *reconstruction* that was
 * faithful; as an *app* it was indefensible. Light and dark are a user setting and an
 * accessibility one, and no four-tab app should flash white when you press the third tab at 3 a.m.
 * The concept's contrast is preserved instead through surface roles, which is where it belongs.
 *
 * ## Colour source
 *
 * Wallpaper colour is the default from API 31 up. Below that, or with it switched off, the app
 * falls back to one of the [Palette] entries.
 */
@Composable
fun ClockTheme(
    settings: ThemeSettings = ThemeSettings(),
    content: @Composable () -> Unit,
) {
    val dark = when (settings.darkMode) {
        DarkMode.SYSTEM -> isSystemInDarkTheme()
        DarkMode.LIGHT -> false
        DarkMode.DARK -> true
    }

    val context = LocalContext.current
    val base = when {
        settings.dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (dark) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        dark -> settings.palette.dark
        else -> settings.palette.light
    }
    val amoled = dark && settings.amoledBlack
    val scheme = remember(base, amoled) { if (amoled) base.toAmoledBlack() else base }

    val activity = LocalActivity.current as? ComponentActivity
    // AndroidX keeps its first configuration listener. Its detector must read the latest
    // app override, not capture the system/default theme from before DataStore loads.
    val currentDark = rememberUpdatedState(dark)
    LaunchedEffect(dark, activity) {
        val bar = SystemBarStyle.auto(AndroidColor.TRANSPARENT, AndroidColor.TRANSPARENT) {
            currentDark.value
        }
        activity?.enableEdgeToEdge(statusBarStyle = bar, navigationBarStyle = bar)
        activity?.window?.let { window ->
            // enableEdgeToEdge owns decor fitting and transparent system-bar colors.
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                window.isNavigationBarContrastEnforced = false
                // Preserve the legacy three-button appearance; Android 15 no longer
                // needs this deprecated divider-color API.
                if (Build.VERSION.SDK_INT < Build.VERSION_CODES.VANILLA_ICE_CREAM) {
                    @Suppress("DEPRECATION")
                    window.navigationBarDividerColor = AndroidColor.TRANSPARENT
                }
            }
        }
    }

    DisposableEffect(activity) {
        val window = activity?.window
        val decor = window?.decorView
        // Splash dismissal and returning from another window can restore platform appearance.
        // Apply on focus without polling or changing the user's chosen theme.
        val listener = ViewTreeObserver.OnWindowFocusChangeListener { focused ->
            if (focused && window != null && decor != null) {
                WindowCompat.getInsetsController(window, decor).apply {
                    isAppearanceLightStatusBars = !currentDark.value
                    isAppearanceLightNavigationBars = !currentDark.value
                }
            }
        }
        decor?.viewTreeObserver?.addOnWindowFocusChangeListener(listener)
        onDispose {
            decor?.viewTreeObserver?.takeIf { it.isAlive }?.removeOnWindowFocusChangeListener(listener)
        }
    }

    CompositionLocalProvider(
        // Without this, `Surface` composites `surfaceTint` over its container as elevation rises,
        // quietly painting grey back over a pure-black AMOLED background, which is the one thing
        // the setting exists to prevent.
        LocalTonalElevationEnabled provides !amoled,
    ) {
        MaterialExpressiveTheme(
            colorScheme = scheme,
            // [M3E-NEW] Under-damped spatial springs paired with critically damped effects
            // springs. Movement overshoots; colour never does. On a clock that asymmetry has more
            // to do than usual, since every screen here has something in continuous motion.
            motionScheme = MotionScheme.expressive(),
            typography = remember { clockTypography() },
            content = content,
        )
    }
}

/**
 * True-black variant of a dark scheme, for OLED panels.
 *
 * A clock is the app most likely to be showing on a dimmed screen at 3 a.m., which is exactly when
 * a near-black surface still glows. Accent roles and the upper containers are untouched: the alarm
 * tiles live on `surfaceContainer`, so blackening the page increases their separation rather than
 * flattening it.
 */
fun ColorScheme.toAmoledBlack(): ColorScheme = copy(
    background = Color.Black,
    surface = Color.Black,
    surfaceDim = Color.Black,
    surfaceContainerLowest = Color.Black,
    surfaceContainerLow = Color(0xFF0A0A0A),
    surfaceContainer = Color(0xFF141018),
    scrim = Color.Black,
)
```

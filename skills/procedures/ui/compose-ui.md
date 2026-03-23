# Compose UI

## Theme System

Access via `MeAppTheme` (aliased as `MeTheme` internally):

```kotlin
MeAppTheme.colorScheme.*    // 50+ semantic colors
MeAppTheme.typography.*     // heading1-6, subHeading1-2, body1-5, link1-2, button1-2
MeAppTheme.spacing.*        // x6s(0.5dp) x4s(2) x3s(4) x2s(6) xs(8) sm(16) md(24) lg(32) xl(40) x2l(48) ...
MeAppTheme.animation.*      // Duration and easing tokens
MeAppTheme.borderRadius.*   // xs, sm, md, lg, xl
```

### Color Examples

```kotlin
MeAppTheme.colorScheme.primaryBackground
MeAppTheme.colorScheme.primaryAction
MeAppTheme.colorScheme.primaryActionDisabled
MeAppTheme.colorScheme.textHeading
MeAppTheme.colorScheme.textBody
MeAppTheme.colorScheme.textError
MeAppTheme.colorScheme.success
MeAppTheme.colorScheme.goal
MeAppTheme.colorScheme.inverse
```

### Typography Examples

```kotlin
MeAppTheme.typography.heading1   // Largest heading
MeAppTheme.typography.body1      // Primary body text
MeAppTheme.typography.button1    // Button labels
MeAppTheme.typography.link1      // Clickable text
```

### Spacing Scale (8-point base)

```
x6s=0.5dp  x4s=2dp  x3s=4dp  x2s=6dp  xs=8dp
sm=16dp    md=24dp  lg=32dp  xl=40dp  x2l=48dp
x3l=56dp   x4l=64dp x5l=72dp x6l=80dp
```

## Previews

**Every composable file must have previews.**

```kotlin
@PreviewTheme
@Composable
fun FooScreenPreview() {
    MeAppTheme {
        FooContent(
            state = FooState(items = listOf(sampleItem)),
            onIntent = {}
        )
    }
}
```

Rules:
- Always `@PreviewTheme` annotation (provides light + dark)
- Always wrap in `MeAppTheme { }`
- Use realistic sample data
- Preview the stateless `Content` composable, not the Screen (avoids ViewModel)

## One Composable Per File

- Each `.kt` file has ONE public composable
- Multiple `@Preview` functions in the same file are OK
- Exception: small private helper composables used only by the main one

## Shared Components (features/common/)

**Always check `features/common/components/` before creating new composables.** 76+ shared components exist:

| Component | Purpose |
|-----------|---------|
| `AppScaffold` | Standard screen scaffold with title, nav icon, actions |
| `AppButton` | Themed button with `ButtonType` and `ButtonSize` |
| `AppInput` | Text input with validation, IME actions |
| `AppIconButton` | Icon-only button |
| `AppText` | Themed text |
| `AppIcon` | Themed icon |
| `AppLoader` | Loading indicator |
| `AppStyledCard` | Themed card container |
| `AppRadioGroup` | Radio button group |
| `AppProfileAvatar` | User avatar display |
| `DialogQueueHost` | Global dialog renderer |
| `ToastCard` | Toast notification card |
| `SettingSection` | Settings list section |
| `BaseListItem` | Standard list row |

Prefer these over raw Material3 components.

## Static Strings

All user-visible text goes in feature-specific `strings/` subfolder:

```kotlin
// features/goal/strings/GoalStrings.kt
object GoalStrings {
    const val PageTitle = "Goal Setting"
    const val SaveButton = "Save"
    const val SaveErrorMessage = "Failed to save goal"
    const val LoaderMessage = "Saving..."
    const val SuccessTitle = "Success!"
    const val SuccessMessage = "Goal Saved."
}
```

Rules:
- PascalCase object name: `<Feature>Strings`
- PascalCase `const val` properties
- Never hardcode text in composables (exception: `@PreviewTheme` sample data)

## AppInput & IME Actions

Every `AppInput` must configure IME behavior with `FocusRequester`:

```kotlin
val emailFocus = remember { FocusRequester() }
val passwordFocus = remember { FocusRequester() }

AppInput(
    value = state.email,
    onValueChange = { onIntent(FooIntent.SetEmail(it)) },
    label = FooStrings.EmailLabel,
    imeAction = ImeAction.Next,
    onImeAction = { passwordFocus.requestFocus() },
    modifier = Modifier.focusRequester(emailFocus),
)

AppInput(
    value = state.password,
    onValueChange = { onIntent(FooIntent.SetPassword(it)) },
    label = FooStrings.PasswordLabel,
    imeAction = ImeAction.Done,
    onImeAction = { onIntent(FooIntent.Submit) },
    modifier = Modifier.focusRequester(passwordFocus),
)
```

Rules:
- `imeAction` — `Next` for intermediate fields, `Done` for last field
- `onImeAction` — moves focus to next field or triggers submit
- Each input gets its own `FocusRequester`

## LazyColumn / LazyRow

All lazy layouts **MUST** have stable `key` parameters. Without keys, Compose recomposes all visible items on any list change.

```kotlin
LazyColumn {
    items(
        items = state.items,
        key = { item -> item.id },  // REQUIRED — stable unique key
    ) { item ->
        FooItemCard(item = item)
    }

    // Headers/footers also need keys
    item(key = "header") { SectionHeader() }
    item(key = "footer") { LoadMoreButton() }
}
```

Rules:
- Key must be a **stable, unique identifier** (e.g., `item.id`, `item.sku`)
- Applies to `LazyColumn`, `LazyRow`, `LazyVerticalGrid`, `LazyHorizontalGrid`
- String keys for static items (headers, footers)
- When passing lists to composable params, prefer `ImmutableList<T>` type for Compose stability

## Screen Structure Pattern

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun FooScreen() {
    val viewModel: FooViewModel = hiltViewModel()
    val state by viewModel.state.collectAsStateWithLifecycle()
    BackHandler { viewModel.handleIntent(FooIntent.OnBack) }
    FooContent(state = state, onIntent = viewModel::handleIntent)
}

@Composable
private fun FooContent(state: FooState, onIntent: (FooIntent) -> Unit) {
    val spacing = MeAppTheme.spacing

    AppScaffold(
        title = FooStrings.PageTitle,
        navigationIcon = {
            AppIconButton(AppIcons.Default.Close) { onIntent(FooIntent.OnBack) }
        },
    ) { scaffoldModifier ->
        Column(
            modifier = scaffoldModifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState()),
        ) {
            // Content here using MeAppTheme tokens
        }
    }
}
```

## Drawable Assets

Before adding new drawables:
1. Check `res/drawable/` for existing assets
2. Naming: `ic_feature_name.xml` for icons, `brand_logo.xml` for brands
3. Prefer vector drawables (`.xml`) over raster images

## Form Validation

Use `FormGroup` + `FormControl` for validated forms:

```kotlin
data class FooFormControls(
    val email: FormControl<String>,
    val password: FormControl<String>,
) {
    companion object {
        fun create() = FooFormControls(
            email = FormControl("", validators = listOf(FormValidations.required(), FormValidations.email())),
            password = FormControl("", validators = listOf(FormValidations.required(), FormValidations.minLength(8))),
        )
    }
}
```

State holds `FormGroup<FooFormControls>`. Check `form.isDirty` and `form.controls.isValid()`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoded color `Color(0xFF...)` | Use `MeAppTheme.colorScheme.*` |
| Hardcoded `16.dp` spacing | Use `MeAppTheme.spacing.sm` |
| Missing `@PreviewTheme` | Add it — ensures light + dark coverage |
| Preview without `MeAppTheme { }` | Wrap content — colors/typography won't resolve otherwise |
| String literal in composable | Move to `<Feature>Strings` object |
| Creating a custom button | Check `AppButton` in common first |
| Missing `imeAction` on AppInput | Always set `imeAction` + `onImeAction` |
| `collectAsState()` | Use `collectAsStateWithLifecycle()` — lifecycle-aware |
| `LazyColumn` without `key` | Add `key = { item -> item.id }` — required for performance |
| `List<T>` in composable params | Prefer `ImmutableList<T>` for Compose stability |

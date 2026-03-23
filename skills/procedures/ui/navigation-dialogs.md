# Navigation & Dialogs

## Navigation

### Routes

All routes defined as sealed classes in `core/navigation/AppRoute.kt`:

```kotlin
sealed class AppRoute : NavKey {
    sealed class Init : AppRoute(), PublicRoute {
        data object Loading : Init()
    }
    sealed class Main : AppRoute() {
        data object Dashboard : Main()
        data object Entry : Main()
        data object History : Main()
        data object Settings : Main()
    }
    sealed class Auth : AppRoute(), PublicRoute {
        data object Landing : Auth()
        data class Login(val email: String? = null) : Auth()
        data object Signup : Auth()
    }
    sealed class AccountSettings : AppRoute() {
        data object ChangePassword : AccountSettings()
        data object Profile : AccountSettings()
        data class ScaleDetails(val scaleId: String) : AccountSettings()
        // ...
    }
    sealed class ScaleSetup : AppRoute() { ... }
    sealed class Integration : AppRoute() { ... }
    sealed class Feed : AppRoute() { ... }
    sealed class Dashboard : AppRoute() { ... }
}
```

### Usage (from any ViewModel)

```kotlin
// Navigate forward
navigationService.navigateTo(AppRoute.AccountSettings.Profile)
navigationService.navigateTo(AppRoute.AccountSettings.ScaleDetails(scaleId = "abc123"))

// Navigate back
navigationService.navigateBack()
```

`navigationService` is inherited from `BaseViewModel` — **never inject it directly**.

### Adding a New Route

1. Add to `AppRoute.kt` as a sealed class/object
2. Register in the navigation host (where routes map to composables)
3. Navigate via `navigationService.navigateTo()`

## Dialogs

`dialogQueueService` is inherited from `BaseViewModel`. Priority-queue based — dialogs show in order.

### Alert Dialog

```kotlin
dialogQueueService.enqueue(
    DialogModel.Alert(
        title = "Error",
        message = "Something went wrong",
        dismissText = "OK",
        onDismiss = { /* optional callback */ },
    )
)
```

### Confirm Dialog

```kotlin
dialogQueueService.enqueue(
    DialogModel.Confirm(
        title = "Delete Entry?",
        message = "This cannot be undone.",
        confirmText = "Delete",
        cancelText = "Cancel",
        primaryActionType = ButtonType.InlineTextPrimary,
        onConfirm = {
            performDelete()
            dialogQueueService.dismissCurrent()
        },
        onCancel = {
            dialogQueueService.dismissCurrent()
        },
    )
)
```

### Custom Dialog

```kotlin
dialogQueueService.enqueue(
    DialogModel.Custom(
        contentKey = DialogType.SomeCustomDialog,
        params = mapOf("key" to value),
        onDismiss = { },
        onConfirm = { result -> },
    )
)
```

## Toasts

```kotlin
dialogQueueService.showToast(
    Toast(
        title = "Success!",       // optional
        message = "Entry saved.",
        action = null,            // optional ActionButton
    )
)
```

With action button:
```kotlin
dialogQueueService.showToast(
    Toast(
        message = "Entry deleted",
        action = ActionButton(text = "Undo", onClick = { undoDelete() }),
    )
)
```

## Loaders

```kotlin
// Show
dialogQueueService.showLoader(message = "Saving...")

// With style
dialogQueueService.showLoader(message = "Loading...", style = LoaderStyle.CIRCULAR)

// Dismiss
dialogQueueService.dismissLoader()
```

Always dismiss loaders in both success and error paths — use try/finally:

```kotlin
dialogQueueService.showLoader(message = "Saving...")
viewModelScope.launch {
    try {
        service.save(data)
        dialogQueueService.dismissLoader()
        dialogQueueService.showToast(Toast(message = "Saved"))
        navigationService.navigateBack()
    } catch (e: Exception) {
        dialogQueueService.dismissLoader()
        handleIntent(FooIntent.SetError(e.message ?: "Failed"))
    }
}
```

## Unsaved Changes Pattern

Standard back-with-unsaved-changes handling:

```kotlin
FooIntent.OnBack -> {
    if (state.value.form.isDirty) {
        dialogQueueService.enqueue(
            DialogModel.Confirm(
                title = "Unsaved Changes",
                message = "Discard changes?",
                confirmText = "Discard",
                cancelText = "Keep Editing",
                onConfirm = {
                    navigationService.navigateBack()
                    dialogQueueService.dismissCurrent()
                },
                onCancel = { dialogQueueService.dismissCurrent() },
            )
        )
    } else {
        navigationService.navigateBack()
    }
}
```

## In-App Browser

```kotlin
// Opens URL in Chrome Custom Tab
openInAppBrowser(url = "https://example.com")
```

Available via `BaseViewModel.openInAppBrowser()` which uses `customTabManager`.

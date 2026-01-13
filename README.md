# Focus Traps in Compose
"Focus Traps" in TextFields usually fall into two categories: the Navigation Trap (where the Tab/Next key inserts whitespace instead of moving focus) and the Ghost Input Trap (where typing stops updating the UI).
Based on the provided documentation, "Focus Traps" in TextFields usually fall into two categories: the **Navigation Trap** (where the Tab/Next key inserts whitespace instead of moving focus) and the **Ghost Input Trap** (where typing stops updating the UI).

Here is how to fix both using the **2026 Best Practices** and **Legacy** methods.

### 1. Fixing the Navigation Trap (The "Tab" Key Issue)
**The Problem:** By default, a standard `TextField` assumes multi-line input. When a user presses **Tab** or **Enter**, the component consumes the event to insert a tab character or newline (`\n`) rather than triggering the focus manager to move to the next field,.

**The Fix:** You must explicitly constrain the field to a single line and define the keyboard action.

**Code Solution (Legacy & Standard M3):**
```kotlin
OutlinedTextField(
    value = text,
    onValueChange = { text = it },
    // Fix 1: Explicitly set singleLine to true to prevent \n insertion
    singleLine = true, 
    // Fix 2: Constrain maxLines to 1 (Redundant if singleLine is true, but safe)
    maxLines = 1, 
    keyboardOptions = KeyboardOptions(
        // Fix 3: Change the Enter key to a 'Next' arrow
        imeAction = ImeAction.Next 
    ),
    keyboardActions = KeyboardActions(
        // Fix 4: Explicitly move focus when the action is triggered
        onNext = { focusManager.moveFocus(FocusDirection.Down) }
    )
)
```


### 2. Fixing the "Ghost Input" Trap (Input Desync)
**The Problem:** In the legacy API (`value` + `onValueChange`), a race condition can occur between the keyboard (IME) and the UI state. If the state update lags or is blocked (e.g., by heavy validation logic), the connection breaks. The keyboard thinks it is typing, but the UI remains empty or "ghosts" the text,.

**The Fix (2026 Best Practice):** Migrate to **`TextFieldState`**. This API uses a synchronous edit buffer that prevents the keyboard and UI from ever getting out of sync,.

**Code Solution (Modern API):**
```kotlin
// 1. Hoist the state (survives recomposition and holds the buffer)
val state = rememberTextFieldState() 

OutlinedTextField(
    state = state, // No 'value' or 'onValueChange' needed
    modifier = Modifier.fillMaxWidth(),
    label = { Text("Username") },
    // 2. Define line limits to prevent the Tab trap automatically
    lineLimits = TextFieldLineLimits.SingleLine, 
    // 3. Handle keyboard actions cleanly
    onKeyboardAction = { 
        focusManager.moveFocus(FocusDirection.Next) 
    }
)
```


### 3. Fixing Traps in Modals & Dialogs
**The Problem:** When a `DropDownMenu` or Dialog closes, the system often "drops" the focus, causing it to reset to the root of the screen. Alternatively, if a `TextField` inside a Dialog doesn't request focus immediately, the user might be trapped navigating the background activity,.

**The Fix:** Use `LaunchedEffect` to request focus *after* the composition is stable.

**Code Solution:**
```kotlin
val focusRequester = remember { FocusRequester() }

Dialog(onDismissRequest = { /*...*/ }) {
    Column {
        TextField(
            modifier = Modifier.focusRequester(focusRequester),
            // ...
        )
    }
    // Fix: Wait for the Dialog to be built, then snap focus to the field
    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
}
```


### Summary Checklist
| Issue | Symptom | Fix |
| :--- | :--- | :--- |
| **Tab Consumption** | Pressing Tab adds space; cursor doesn't move. | Set `singleLine = true` and `ImeAction.Next`. |
| **Ghost Typing** | Keyboard types but text field stays empty. | Migrate to `TextFieldState` or ensure `onValueChange` updates state immediately,. |
| **Modal Trap** | Keyboard navigation stuck in background. | Use `LaunchedEffect` to `requestFocus` inside the Dialog content. |


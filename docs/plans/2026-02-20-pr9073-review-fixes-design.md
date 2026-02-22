# PR #9073 Review Fixes — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Address all review comments from ematipico (changes requested) and arendjr on PR #9073.

**Architecture:** Move plugin text edit application logic from JavaScript-only into `ProcessFixAll` as a shared method. Add CSS/JSON support. Fix naming, imports, and robustness issues. Add comprehensive tests.

**Tech Stack:** Rust, biome_analyze, biome_service, biome_grit_patterns, biome_cli tests, biome_lsp tests

---

### Task 1: Rename `with_actions` to `with_plugin_actions` and fix imports

**Files:**
- Modify: `crates/biome_analyze/src/signals.rs:1-20,129-131`
- Modify: `crates/biome_analyze/src/analyzer_plugin.rs:140-142`

**Step 1: Fix imports in signals.rs**

In `crates/biome_analyze/src/signals.rs`, add `PluginActionData` to the top-level import block (line 5):

```rust
use crate::{
    AnalyzerDiagnostic, AnalyzerOptions, OtherActionCategory, PluginActionData, Queryable,
    RuleDiagnostic, RuleGroup, ServiceBag, SuppressionAction,
    categories::ActionCategory,
    context::RuleContext,
    registry::{RuleLanguage, RuleRoot},
    rule::Rule,
};
```

Then replace all `crate::PluginActionData` with `PluginActionData` in the file (lines ~117 and ~129).

**Step 2: Rename method**

In `crates/biome_analyze/src/signals.rs` line 129, rename:

```rust
pub fn with_plugin_actions(mut self, actions: Vec<PluginActionData>) -> Self {
```

**Step 3: Fix action category**

In `crates/biome_analyze/src/signals.rs` line 165, change:

```rust
category: ActionCategory::QuickFix(Cow::Borrowed("plugin")),
```

**Step 4: Update call site**

In `crates/biome_analyze/src/analyzer_plugin.rs` line 140, change:

```rust
.with_plugin_actions(actions.clone())
```

**Step 5: Run tests to verify**

Run: `cargo t -p biome_analyze`
Expected: PASS

**Step 6: Commit**

```
refactor(biome_analyze): rename with_actions to with_plugin_actions
```

---

### Task 2: Move plugin text edit logic to ProcessFixAll

**Files:**
- Modify: `crates/biome_service/src/file_handlers/mod.rs:847-873`
- Modify: `crates/biome_service/src/file_handlers/javascript.rs:1097-1129`

**Step 1: Add `apply_plugin_text_edit` method to ProcessFixAll**

In `crates/biome_service/src/file_handlers/mod.rs`, add after `record_text_edit_fix` (after line 873):

```rust
    /// Apply a plugin text edit if present and the text actually changed.
    /// Returns `Some(new_text)` if the edit was applied, `None` otherwise.
    pub(crate) fn apply_plugin_text_edit(
        &mut self,
        text_edit: Option<(TextRange, TextEdit)>,
        current_text: &str,
    ) -> Result<Option<String>, WorkspaceError> {
        let Some((range, edit)) = text_edit else {
            return Ok(None);
        };
        let new_text = edit.new_string(current_text);
        if new_text == current_text {
            return Ok(None);
        }
        self.record_text_edit_fix(
            range,
            new_text.len() as u32,
            Some(("plugin", "gritql")),
        )?;
        Ok(Some(new_text))
    }
```

Add `use biome_text_edit::TextEdit;` to the imports at the top of `mod.rs` if not already present.

**Step 2: Simplify javascript.rs plugin text edit handling**

In `crates/biome_service/src/file_handlers/javascript.rs`, replace lines 1109-1129 (the `if let Some((range, edit)) = plugin_text_edit` block) with:

```rust
            if let Some(new_text) = process_fix_all.apply_plugin_text_edit(
                plugin_text_edit,
                &tree.syntax().to_string(),
            )? {
                let options = params.settings.parse_options::<JsLanguage>(
                    params.biome_path,
                    &params.document_file_source,
                );
                let parse = biome_js_parser::parse(&new_text, file_source, options);
                tree = parse.tree();
                continue;
            }
```

**Step 3: Run tests**

Run: `cargo t -p biome_cli check_plugin`
Expected: All 3 existing plugin rewrite tests PASS

**Step 4: Commit**

```
refactor(biome_service): move plugin text edit logic to ProcessFixAll
```

---

### Task 3: Add plugin rewrite support for CSS and JSON

**Files:**
- Modify: `crates/biome_service/src/file_handlers/css.rs:720-752`
- Modify: `crates/biome_service/src/file_handlers/json.rs:715-742`

**Step 1: Add plugin text edit handling to CSS fix_all**

In `crates/biome_service/src/file_handlers/css.rs`, in the `fix_all` function loop, add text edit extraction before `process_action` (before the `let result = process_fix_all.process_action(action, ...)` line):

```rust
        let plugin_text_edit = action.as_ref().and_then(|a| a.text_edit.clone());
```

Then in the `if result.is_none()` block, before the `return process_fix_all.finish(...)`, add:

```rust
            if let Some(new_text) = process_fix_all.apply_plugin_text_edit(
                plugin_text_edit,
                &tree.syntax().to_string(),
            )? {
                let parse = biome_css_parser::parse_css(
                    &new_text,
                    biome_css_parser::CssParserOptions::default(),
                );
                tree = parse.tree();
                continue;
            }
```

**Step 2: Add plugin text edit handling to JSON fix_all**

Same pattern in `crates/biome_service/src/file_handlers/json.rs`:

Before `process_action`:
```rust
        let plugin_text_edit = action.as_ref().and_then(|a| a.text_edit.clone());
```

In `if result.is_none()` block, before `return process_fix_all.finish(...)`:
```rust
            if let Some(new_text) = process_fix_all.apply_plugin_text_edit(
                plugin_text_edit,
                &tree.syntax().to_string(),
            )? {
                let parse = biome_json_parser::parse_json(
                    &new_text,
                    biome_json_parser::JsonParserOptions::default(),
                );
                tree = parse.tree();
                continue;
            }
```

**Step 3: Run tests**

Run: `cargo t -p biome_service`
Expected: PASS

**Step 4: Commit**

```
feat(biome_service): support plugin rewrites for CSS and JSON
```

---

### Task 4: Replace debug_assert with hard error for overlapping replacements

**Files:**
- Modify: `crates/biome_grit_patterns/src/linearization.rs:85-93`

**Step 1: Replace debug_assert**

In `crates/biome_grit_patterns/src/linearization.rs`, replace lines 85-93:

```rust
        if *start < cursor {
            return Err(GritPatternError::new(format!(
                "overlapping replacements detected: start={start}, cursor={cursor}"
            )));
        }
        if *start > cursor {
            result.push_str(&source[cursor..*start]);
        }
```

Remove the `debug_assert!` that was there before.

**Step 2: Run tests**

Run: `cargo t -p biome_grit_patterns`
Expected: PASS

**Step 3: Commit**

```
fix(biome_grit_patterns): return error for overlapping replacements
```

---

### Task 5: Add `--write` only test for plugin rewrites

**Files:**
- Modify: `crates/biome_cli/tests/commands/check.rs` (add test near line 3535)

**Step 1: Write the test**

Add a new test function after `check_plugin_rewrite_no_write`:

```rust
#[test]
fn check_plugin_rewrite_write_without_unsafe() {
    let mut console = BufferConsole::default();
    let fs = MemoryFileSystem::default();

    let config = r#"{
        "plugins": ["useConsoleInfo.grit"],
        "linter": {
            "rules": { "all": { "recommended": false } }
        }
    }"#;

    let plugin = r#"language js

`console.log($msg)` as $call where {
    register_diagnostic(
        span = $call,
        message = "Use console.info instead of console.log.",
        severity = "warning"
    ),
    $call => `console.info($msg)`
}"#;

    let file_path = Utf8Path::new("file.js");
    fs.insert(file_path.into(), "console.log(\"hello\");\n");
    fs.insert(Utf8Path::new("biome.json").into(), config);
    fs.insert(Utf8Path::new("useConsoleInfo.grit").into(), plugin);

    let (fs, result) = run_cli_with_server_workspace(
        fs,
        &mut console,
        Args::from(["check", "--write", file_path.as_str()].as_slice()),
    );

    // --write without --unsafe should NOT apply unsafe fixes
    assert_file_contents(&fs, file_path, "console.log(\"hello\");\n");
    assert_cli_snapshot(SnapshotPayload::new(
        module_path!(),
        "check_plugin_rewrite_write_without_unsafe",
        fs,
        console,
        result,
    ));
}
```

**Step 2: Run test to see it fail (snapshot missing)**

Run: `cargo t -p biome_cli check_plugin_rewrite_write_without_unsafe`
Expected: FAIL (missing snapshot)

**Step 3: Review and accept snapshot**

Run: `cargo insta accept`
Review that the snapshot shows the unsafe fix suggestion.

**Step 4: Run test again to verify**

Run: `cargo t -p biome_cli check_plugin_rewrite_write_without_unsafe`
Expected: PASS

**Step 5: Commit**

```
test(biome_cli): add --write only test for plugin rewrites
```

---

### Task 6: Add CSS and JSON plugin rewrite tests

**Files:**
- Modify: `crates/biome_cli/tests/commands/check.rs` (add 2 tests)

**Step 1: Write CSS plugin rewrite test**

```rust
#[test]
fn check_plugin_apply_rewrite_css() {
    let mut console = BufferConsole::default();
    let fs = MemoryFileSystem::default();

    let config = r#"{
        "plugins": ["banRed.grit"],
        "css": { "linter": { "enabled": true } },
        "linter": {
            "rules": { "all": { "recommended": false } }
        }
    }"#;

    let plugin = r#"language css

`red` as $color where {
    register_diagnostic(
        span = $color,
        message = "Avoid using red.",
        severity = "warning"
    ),
    $color => `blue`
}"#;

    let file_path = Utf8Path::new("file.css");
    fs.insert(file_path.into(), "a { color: red; }\n");
    fs.insert(Utf8Path::new("biome.json").into(), config);
    fs.insert(Utf8Path::new("banRed.grit").into(), plugin);

    let (fs, result) = run_cli_with_server_workspace(
        fs,
        &mut console,
        Args::from(["check", "--write", "--unsafe", file_path.as_str()].as_slice()),
    );

    assert!(result.is_ok(), "run_cli returned {result:?}");
    assert_file_contents(&fs, file_path, "a { color: blue; }\n");
    assert_cli_snapshot(SnapshotPayload::new(
        module_path!(),
        "check_plugin_apply_rewrite_css",
        fs,
        console,
        result,
    ));
}
```

**Step 2: Write JSON plugin rewrite test**

```rust
#[test]
fn check_plugin_apply_rewrite_json() {
    let mut console = BufferConsole::default();
    let fs = MemoryFileSystem::default();

    let config = r#"{
        "plugins": ["fixVersion.grit"],
        "json": { "linter": { "enabled": true } },
        "linter": {
            "rules": { "all": { "recommended": false } }
        }
    }"#;

    let plugin = r#"language json

`"1.0.0"` as $version where {
    register_diagnostic(
        span = $version,
        message = "Update version.",
        severity = "warning"
    ),
    $version => `"2.0.0"`
}"#;

    let file_path = Utf8Path::new("package.json");
    fs.insert(file_path.into(), r#"{"version": "1.0.0"}"#.to_string() + "\n");
    fs.insert(Utf8Path::new("biome.json").into(), config);
    fs.insert(Utf8Path::new("fixVersion.grit").into(), plugin);

    let (fs, result) = run_cli_with_server_workspace(
        fs,
        &mut console,
        Args::from(["check", "--write", "--unsafe", file_path.as_str()].as_slice()),
    );

    assert!(result.is_ok(), "run_cli returned {result:?}");
    assert_file_contents(&fs, file_path, "{\"version\": \"2.0.0\"}\n");
    assert_cli_snapshot(SnapshotPayload::new(
        module_path!(),
        "check_plugin_apply_rewrite_json",
        fs,
        console,
        result,
    ));
}
```

**Step 3: Run tests, accept snapshots**

Run: `cargo t -p biome_cli check_plugin_apply_rewrite_css check_plugin_apply_rewrite_json`
Expected: FAIL (missing snapshots)
Run: `cargo insta accept`
Run tests again: PASS

**Step 4: Commit**

```
test(biome_cli): add CSS and JSON plugin rewrite tests
```

---

### Task 7: Add LSP test for plugin diagnostics

**Files:**
- Modify: `crates/biome_lsp/src/server.tests.rs` (add test near line 1842)

**Step 1: Write LSP diagnostic test for plugin rewrite**

Add after `plugin_load_error_show_message` test:

```rust
#[tokio::test]
async fn plugin_rewrite_pull_diagnostics() -> Result<()> {
    let fs = MemoryFileSystem::default();

    let config = r#"{
        "plugins": ["useConsoleInfo.grit"],
        "linter": {
            "rules": { "all": { "recommended": false } }
        }
    }"#;

    let plugin = br#"language js

`console.log($msg)` as $call where {
    register_diagnostic(
        span = $call,
        message = "Use console.info instead of console.log.",
        severity = "warning"
    ),
    $call => `console.info($msg)`
}"#;

    fs.insert(to_utf8_file_path_buf(uri!("biome.json")), config);
    fs.insert(to_utf8_file_path_buf(uri!("useConsoleInfo.grit")), plugin);

    let factory = ServerFactory::new_with_fs(Arc::new(fs));
    let (service, client) = factory.create().into_inner();

    let (stream, sink) = client.split();
    let mut server = Server::new(service);

    let (sender, mut receiver) = channel(CHANNEL_BUFFER_SIZE);
    let reader = tokio::spawn(client_handler(stream, sink, sender));

    server.initialize().await?;
    server.initialized().await?;

    server.load_configuration().await?;

    server
        .open_named_document(
            "console.log(\"hello\");",
            uri!("document.js"),
            "javascript",
        )
        .await?;

    let notification =
        wait_for_notification(&mut receiver, |n| n.is_publish_diagnostics()).await;

    // Verify that the plugin diagnostic is emitted
    if let Some(ServerNotification::PublishDiagnostics(params)) = &notification {
        assert_eq!(params.diagnostics.len(), 1);
        let diag = &params.diagnostics[0];
        assert_eq!(diag.severity, Some(lsp::DiagnosticSeverity::WARNING));
        assert!(diag.message.contains("console.info"));
    } else {
        panic!("Expected PublishDiagnostics, got {notification:?}");
    }

    server.close_document().await?;
    server.shutdown().await?;
    reader.abort();

    Ok(())
}
```

**Step 2: Run test**

Run: `cargo t -p biome_lsp plugin_rewrite_pull_diagnostics`
Expected: PASS

**Step 3: Commit**

```
test(biome_lsp): add test for plugin rewrite diagnostics
```

---

### Task 8: PR housekeeping (no commit)

**Step 1: Retarget PR to `next` branch**

Run: `gh pr edit 9073 --base next`

Note: This requires the `next` branch to exist. Check with maintainer if needed.

**Step 2: Update changeset**

In `.changeset/fixable-gritql-plugins.md`, update to clarify all fixes are unsafe:

```markdown
---
"@biomejs/biome": minor
---

Added support for applying GritQL plugin rewrites as code actions. GritQL plugins that use the rewrite operator (`=>`) now produce fixable diagnostics. All plugin rewrites are treated as unsafe fixes and require `--write --unsafe` to apply.
```

**Step 3: Reply to ematipico's question on the changeset**

Reply on the PR inline comment ("Are ALL plugin fixes unsafe?"):

> Yes, for this initial implementation all plugin fixes are treated as unsafe (Applicability::MaybeIncorrect). GritQL rewrites are arbitrary text transforms and we can't verify semantic correctness. A follow-up PR will add an API for plugin authors to declare fix safety.

**Step 4: Reply to ematipico's top-level review**

Reply addressing all items with commit references.

**Step 5: Restore PR template**

Add the missing documentation section to the PR body. Note that a docs PR to `biomejs/website` against `next` is needed.

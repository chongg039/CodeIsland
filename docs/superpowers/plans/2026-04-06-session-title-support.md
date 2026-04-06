# Session Title Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make session cards show real provider session titles for Codex, keep session IDs accessible and copyable, and create a provider-agnostic title abstraction for future tools such as Claude Code.

**Architecture:** Add a shared session-title model to session state, populate Codex titles from `~/.codex/session_index.jsonl`, and make the UI read one provider-agnostic title surface instead of deriving the header directly from `cwd`. Keep project/folder context as secondary information and replace the old duplicate-only short-ID suffix with a dedicated session-ID control.

**Tech Stack:** Swift 5.9, SwiftUI, Swift Package Manager, XCTest

---

## File Structure

- Modify: `Package.swift`
  Add test targets so the new parser/model behavior can be covered by automated tests.
- Modify: `Sources/CodeIslandCore/SessionSnapshot.swift`
  Add provider-agnostic title fields and computed display helpers.
- Create: `Sources/CodeIsland/SessionTitleStore.swift`
  Centralize provider title lookup starting with Codex's `session_index.jsonl`.
- Modify: `Sources/CodeIsland/AppState.swift`
  Resolve and refresh provider titles during Codex session discovery and event handling.
- Modify: `Sources/CodeIsland/NotchPanelView.swift`
  Use title-first rendering, demote project name to secondary text, and add a copyable session-ID control.
- Modify: `Sources/CodeIsland/L10n.swift`
  Add minimal copy/session-ID strings if the UI needs button labels or tooltips.
- Create: `Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift`
  Cover title/fallback/project-display helper behavior.
- Create: `Tests/CodeIslandTests/SessionTitleStoreTests.swift`
  Cover Codex session-index parsing and latest-title selection.

### Task 1: Add test scaffolding and title-model coverage

**Files:**
- Modify: `Package.swift`
- Create: `Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift`

- [ ] **Step 1: Write the failing core title-behavior tests**

```swift
import XCTest
@testable import CodeIslandCore

final class SessionSnapshotTitleTests: XCTestCase {
    func testDisplayTitlePrefersProviderSessionTitle() {
        var snapshot = SessionSnapshot()
        snapshot.sessionTitle = "Investigate icon sizing"

        XCTAssertEqual(snapshot.displayTitle(sessionId: "019d6331-3593-7b53-9513-c1dd25d708b0"),
                       "Investigate icon sizing")
    }

    func testDisplayTitleFallsBackToSessionIdWhenNoProviderTitleExists() {
        let snapshot = SessionSnapshot()

        XCTAssertEqual(snapshot.displayTitle(sessionId: "019d632b-abee-76e3-80d6-667ea86ebeaf"),
                       "019d632b-abee-76e3-80d6-667ea86ebeaf")
    }

    func testProjectDisplayNameStillUsesFolderName() {
        var snapshot = SessionSnapshot()
        snapshot.cwd = "/Users/wangnov/CodeIsland"

        XCTAssertEqual(snapshot.projectDisplayName, "CodeIsland")
    }
}
```

- [ ] **Step 2: Run the new core test target and verify it fails**

Run: `swift test --filter SessionSnapshotTitleTests`
Expected: FAIL because test targets and title helpers do not exist yet.

- [ ] **Step 3: Add the minimal test target plumbing and title fields**

```swift
// Package.swift
.testTarget(
    name: "CodeIslandCoreTests",
    dependencies: ["CodeIslandCore"]
),
```

```swift
// SessionSnapshot.swift
public enum SessionTitleSource {
    case codexThreadName
}

public var sessionTitle: String?
public var sessionTitleSource: SessionTitleSource?

public func displayTitle(sessionId: String) -> String {
    if let sessionTitle, !sessionTitle.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty {
        return sessionTitle
    }
    return sessionId
}

public var projectDisplayName: String {
    displayName
}
```

- [ ] **Step 4: Run the new core test target and verify it passes**

Run: `swift test --filter SessionSnapshotTitleTests`
Expected: PASS

- [ ] **Step 5: Commit the scaffolding task**

```bash
git add Package.swift Sources/CodeIslandCore/SessionSnapshot.swift Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift
git commit -m "test: add session title model coverage"
```

### Task 2: Add Codex session-title lookup

**Files:**
- Create: `Sources/CodeIsland/SessionTitleStore.swift`
- Modify: `Package.swift`
- Create: `Tests/CodeIslandTests/SessionTitleStoreTests.swift`

- [ ] **Step 1: Write the failing Codex title-store tests**

```swift
import XCTest
@testable import CodeIsland

final class SessionTitleStoreTests: XCTestCase {
    func testCodexThreadNameLookupReturnsLatestMatchingTitle() throws {
        let lines = [
            #"{"id":"019d6330-beed-7a13-b61e-cacf03d3cefe","thread_name":"Old title","updated_at":"2026-04-06T14:20:00Z"}"#,
            #"{"id":"019d6330-beed-7a13-b61e-cacf03d3cefe","thread_name":"充分探索项目找到通用问题","updated_at":"2026-04-06T14:28:21Z"}"#
        ].joined(separator: "\n")

        let title = try SessionTitleStore.codexThreadName(
            sessionId: "019d6330-beed-7a13-b61e-cacf03d3cefe",
            indexContents: lines
        )

        XCTAssertEqual(title, "充分探索项目找到通用问题")
    }

    func testCodexThreadNameLookupIgnoresBlankTitlesAndBadLines() throws {
        let lines = [
            #"{"id":"019d6331-3593-7b53-9513-c1dd25d708b0","thread_name":"","updated_at":"2026-04-06T14:28:38Z"}"#,
            "not-json"
        ].joined(separator: "\n")

        let title = try SessionTitleStore.codexThreadName(
            sessionId: "019d6331-3593-7b53-9513-c1dd25d708b0",
            indexContents: lines
        )

        XCTAssertNil(title)
    }
}
```

- [ ] **Step 2: Run the title-store test target and verify it fails**

Run: `swift test --filter SessionTitleStoreTests`
Expected: FAIL because the store does not exist yet.

- [ ] **Step 3: Implement the smallest Codex title reader**

```swift
import Foundation
import CodeIslandCore

enum SessionTitleStore {
    static func codexThreadName(sessionId: String, indexContents: String) throws -> String? {
        struct Entry: Decodable {
            let id: String
            let thread_name: String?
            let updated_at: String?
        }

        var latest: (Date, String)?
        let iso = ISO8601DateFormatter()

        for line in indexContents.split(separator: "\n") {
            guard let data = line.data(using: .utf8),
                  let entry = try? JSONDecoder().decode(Entry.self, from: data),
                  entry.id == sessionId,
                  let rawTitle = entry.thread_name?.trimmingCharacters(in: .whitespacesAndNewlines),
                  !rawTitle.isEmpty
            else { continue }

            let updated = entry.updated_at.flatMap(iso.date(from:)) ?? .distantPast
            if latest == nil || updated >= latest!.0 {
                latest = (updated, rawTitle)
            }
        }

        return latest?.1
    }

    static func codexThreadName(sessionId: String) -> String? {
        let path = NSHomeDirectory() + "/.codex/session_index.jsonl"
        guard let contents = try? String(contentsOfFile: path, encoding: .utf8) else { return nil }
        return try? codexThreadName(sessionId: sessionId, indexContents: contents)
    }
}
```

- [ ] **Step 4: Run the title-store test target and verify it passes**

Run: `swift test --filter SessionTitleStoreTests`
Expected: PASS

- [ ] **Step 5: Commit the Codex title-store task**

```bash
git add Package.swift Sources/CodeIsland/SessionTitleStore.swift Tests/CodeIslandTests/SessionTitleStoreTests.swift
git commit -m "feat: add codex session title store"
```

### Task 3: Populate session titles in app state

**Files:**
- Modify: `Sources/CodeIsland/AppState.swift`
- Modify: `Sources/CodeIslandCore/SessionSnapshot.swift`
- Test: `Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift`
- Test: `Tests/CodeIslandTests/SessionTitleStoreTests.swift`

- [ ] **Step 1: Write the failing integration test or targeted helper test**

If direct `AppState` testing is too heavy, add a small helper test around "apply provider title to snapshot" behavior instead of forcing a large actor test.

```swift
func testSessionTitleAssignmentDoesNotOverwriteProjectDisplayName() {
    var snapshot = SessionSnapshot()
    snapshot.cwd = "/Users/wangnov/CodeIsland"
    snapshot.sessionTitle = "查看图标bug和窗口大小bug解法"

    XCTAssertEqual(snapshot.displayTitle(sessionId: "019d6331-3593-7b53-9513-c1dd25d708b0"),
                   "查看图标bug和窗口大小bug解法")
    XCTAssertEqual(snapshot.projectDisplayName, "CodeIsland")
}
```

- [ ] **Step 2: Run the relevant test and verify the missing integration behavior fails**

Run: `swift test --filter SessionSnapshotTitleTests`
Expected: FAIL or expose the missing population path before UI integration.

- [ ] **Step 3: Implement title refresh in Codex discovery/event flow**

Add a small helper in `AppState`:

```swift
private func refreshProviderTitle(for sessionId: String) {
    guard let session = sessions[sessionId] else { return }

    switch session.source {
    case "codex":
        sessions[sessionId]?.sessionTitle = SessionTitleStore.codexThreadName(sessionId: sessionId)
        sessions[sessionId]?.sessionTitleSource =
            sessions[sessionId]?.sessionTitle == nil ? nil : .codexThreadName
    default:
        break
    }
}
```

Call it:

- after a Codex session is discovered and inserted
- after Codex hook events update an existing session

- [ ] **Step 4: Run the focused tests and a package build**

Run: `swift test --filter SessionSnapshotTitleTests`
Expected: PASS

Run: `swift test --filter SessionTitleStoreTests`
Expected: PASS

Run: `swift build`
Expected: PASS

- [ ] **Step 5: Commit the state-population task**

```bash
git add Sources/CodeIsland/AppState.swift Sources/CodeIslandCore/SessionSnapshot.swift Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift Tests/CodeIslandTests/SessionTitleStoreTests.swift
git commit -m "feat: populate provider session titles"
```

### Task 4: Update session card UI and session-ID affordance

**Files:**
- Modify: `Sources/CodeIsland/NotchPanelView.swift`
- Modify: `Sources/CodeIsland/L10n.swift`

- [ ] **Step 1: Write the failing UI-facing assertion in the smallest testable unit**

If a full SwiftUI test is not practical, add a pure helper for truncated ID formatting and copy payload.

```swift
func testSessionIdChipTextUsesCompactPrefixButPreservesFullIdForCopy() {
    let fullId = "019d6331-3593-7b53-9513-c1dd25d708b0"

    XCTAssertEqual(SessionIDDisplay.chipText(for: fullId), "#019d...")
    XCTAssertEqual(SessionIDDisplay.copyText(for: fullId), fullId)
}
```

- [ ] **Step 2: Run the focused test and verify it fails**

Run: `swift test --filter SessionIDDisplay`
Expected: FAIL because the helper/control does not exist yet.

- [ ] **Step 3: Implement the card-header update**

Change the header so it uses:

```swift
ProjectNameLink(
    name: session.displayTitle(sessionId: sessionId),
    cwd: session.cwd,
    ...
)
```

Then add secondary project text using `session.projectDisplayName` or `session.subtitle`.

Add a small session-ID control:

```swift
Button {
    NSPasteboard.general.clearContents()
    NSPasteboard.general.setString(sessionId, forType: .string)
} label: {
    Text("#\(sessionId.prefix(4))...")
}
```

Remove the old `showIdSuffix` duplicate-only behavior once the dedicated ID control is present.

- [ ] **Step 4: Run build/tests and manually verify the live UI**

Run: `swift build`
Expected: PASS

Manual check:
- named Codex sessions show thread titles
- unnamed sessions show full session ID as title
- project/folder context still appears as secondary information
- session-ID button copies the full session ID

- [ ] **Step 5: Commit the UI task**

```bash
git add Sources/CodeIsland/NotchPanelView.swift Sources/CodeIsland/L10n.swift
git commit -m "feat: show provider titles in session cards"
```

### Task 5: Final verification

**Files:**
- Modify: any files touched above as needed

- [ ] **Step 1: Run the full automated verification set**

Run: `swift test`
Expected: PASS

Run: `swift build`
Expected: PASS

- [ ] **Step 2: Run a targeted live verification against real Codex state**

Checklist:
- open multiple Codex sessions in the same repo
- confirm named sessions read `thread_name` from `~/.codex/session_index.jsonl`
- confirm the unnamed session still remains distinguishable
- confirm copy button places full session ID on the pasteboard

- [ ] **Step 3: Review git diff for scope control**

Run: `git diff --stat main...HEAD`
Expected: only title-model, Codex resolver, UI, tests, and docs changes

- [ ] **Step 4: Commit any final cleanup**

```bash
git add Package.swift Sources/CodeIslandCore/SessionSnapshot.swift Sources/CodeIsland/AppState.swift Sources/CodeIsland/SessionTitleStore.swift Sources/CodeIsland/NotchPanelView.swift Sources/CodeIsland/L10n.swift Tests/CodeIslandCoreTests/SessionSnapshotTitleTests.swift Tests/CodeIslandTests/SessionTitleStoreTests.swift
git commit -m "test: finalize session title support"
```

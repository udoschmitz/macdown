# MacDown-Fork — Umbauplan: Suche, Zoom, Editor-Fenster

**Zweck dieses Dokuments.** Es ist bewusst zweifach nutzbar:

1. **Für Udo zum Lesen** — was wir warum ändern, in Prosa, ohne dass man den Code kennen muss.
2. **Für Claude als Bau-Spezifikation** — mit `Datei:Zeile`-Ankern, den konkreten API-Aufrufen und einer „Definition of Done" pro Punkt, damit der spätere Bau nicht neu recherchiert werden muss.

---

## ▶ Aktueller Stand & nächster Schritt (zuerst lesen)

**Stand 2026-10-18 — entschieden, aber noch nichts gebaut.**

- **Entscheidung (Udo, 2026-10-18):** Wir machen es **genau in der Reihenfolge aus Abschnitt 5**: Build-Baseline → Zoom (B) → Suche (A) → Udo testet live → erst dann über das Editor-Fenster (C) entscheiden. Für C ist damit Option **(b)** aus Abschnitt 4.3 gewählt.
- **Umgesetzt wird in einer späteren Session**, nicht sofort.
- **Nächster konkreter Schritt:** Abschnitt 7, Schritt 1–6 — das *unveränderte* MacDown bauen und starten. Noch nichts davon ist passiert (geprüft 2026-10-06): kein `Pods/`, kein DerivedData-Build, kein Feature-Branch, Klon steht auf Upstream-Commit `3e2a2bf`.
- **Achtung, Verwechslungsgefahr:** Die App in `/Applications/MacDown.app` ist **nicht** aus diesem Repo gebaut. Sie ist eine per Safari geladene Fremd-Build (Version 1.0.0, signiert von „Developer ID Application: Mass Metal, LLC (L2V57226NH)“, Quarantäne-Flag gesetzt; geprüft 2026-10-06). Unsere eigene Build hat dieselbe Bundle-ID `com.uranusjr.macdown` → beim Testen **immer den Pfad der frisch gebauten App starten** und mit `ps aux | grep MacDown` prüfen, welche Kopie läuft. Wie wir mit der installierten Kopie umgehen (ersetzen? daneben behalten?), ist noch offen — siehe Abschnitt 8.
- **Branch:** Gearbeitet wird auf `feature/reader-search-zoom` (angelegt 2026-10-18, erster Commit = diese Datei). `master` bleibt unverändert auf Upstream `3e2a2bf`.
- **Backup:** Das Repo liegt bewusst außerhalb der Nextcloud und hat noch **kein eigenes Remote** (`origin` = Upstream `MacDownApp/macdown`, dorthin wird nie gepusht). Der Stand existiert also nur auf diesem Mac. Ein eigenes Remote (Udos Gitea, ggf. GitHub-Fork) kommt mit dem Veröffentlichen — siehe Abschnitt 7 unten.

---

**Stand der Recherche:** 2026-08-28. Basis ist ein flacher Klon des offiziellen `MacDownApp/macdown` in `~/Programming/Xcode/macdown`. Alle mit ✅ markierten Aussagen sind **im Quellcode verifiziert** (Datei:Zeile). 🔶 = Design-Empfehlung/Einschätzung, noch nicht gebaut.

---

## 0. Worum es geht (die drei Wünsche)

Udo nutzt MacDown vor allem als **Markdown-Reader**. Drei Dinge am UI stören:

1. **Die Suche (⌘F) sucht im versteckten Quelltext-Editor statt im sichtbaren Vorschau-Text.** Wenn der Editor ausgeblendet ist und man nur Wörter im gerenderten Text finden/hervorheben will, ist das kontraproduktiv. (Für Suchen-und-Ersetzen im Editor ergibt das heutige Verhalten dagegen Sinn.)
2. **Es gibt kein ⌘+ / ⌘− zum Vergrößern des angezeigten Textes**, so wie man es von Websites/Browsern kennt.
3. **Der Editor sollte evtl. ein eigenes Fenster sein** statt einer ausblendbaren Split-Ansicht — das jetzige „Verstecken" fühlt sich falsch an und ist Mit-Ursache von Problem 1.

**Warum forken.** MacDown ist **MIT-lizenziert** und seit ~2021 praktisch schlafend (letzter Commit `3e2a2bf`, 2021-04-29, ein Dependabot-Merge). Es kommen keine Upstream-Fixes mehr; eigene Änderungen müssen wir selbst tragen und können sie als Fork veröffentlichen. Lizenzrechtlich ist der Fork unproblematisch (MIT), die Lizenzdateien der Abhängigkeiten liegen im Verzeichnis `LICENSE/` und werden mitgeführt.

---

## 1. Ist-Zustand — das verifizierte Architektur-Audit

### 1.1 Grundbefund: Die Vorschau ist die alte `WebView` (nicht `WKWebView`)

Das ist die wichtigste Einzeltatsache, weil sie bestimmt, **wie** wir Suche und Zoom bauen.

- ✅ `MPDocument.m:10` — `#import <WebKit/WebKit.h>`
- ✅ `MPDocument.m:191` — `@property (weak) IBOutlet WebView *preview;` → Vorschau ist die **Legacy-`WebView`** (WebKit erster Generation).
- ✅ `MPDocument.m:176` — Dokument implementiert `WebFrameLoadDelegate, WebPolicyDelegate, WebResourceLoadDelegate, WebEditingDelegate` (alte WebView-Delegates; bei `WKWebView` hießen sie `WKNavigationDelegate` etc.).
- ✅ `MPDocument.m:1722` — nutzt `self.preview.mainFrame.javaScriptContext` — **reine WebView-API**, existiert bei `WKWebView` nicht.
- ✅ Verifiziert: `grep -rln WKWebView MacDown/` → **kein Treffer**. `WKWebView` kommt im ganzen Code nicht vor.

**Konsequenz.** `WebView` ist seit macOS 10.14 **deprecated**, funktioniert aber weiter. Für einen selbst-signierten Fork außerhalb des App Store ist das in Ordnung. Siehe die Weggabelung in Abschnitt 6.

### 1.2 Build-Voraussetzungen (verifiziert)

- ✅ **CocoaPods nötig.** `Podfile`, `platform :osx, "10.8"`, 9 Pods. Zuletzt genutzte Version laut `Podfile.lock`: **CocoaPods 1.8.4**. Mehrere Pods stammen aus einem eigenen, gepatchten Spec-Repo `github.com/MacDownApp/cocoapods-specs.git` (hoedown, handlebars-objc, LibYAML) → diese Source muss erreichbar sein. 🔶 Mit sehr neuem CocoaPods/Ruby kann `pod install` wegen der alten Specs zicken — im Zweifel die gelockten Versionen respektieren.
- ✅ **Ein Git-Submodul:** `.gitmodules` → `Dependency/prism` (`github.com/PrismJS/prism.git`). Für den Build vermutlich nötig (Syntax-Highlighting-Assets) → `git submodule update --init`.
- ✅ **Über die Workspace bauen, nicht das Projekt:** `MacDown.xcworkspace` (wegen CocoaPods). `MacDown.xcodeproj` liegt nur daneben.
- ✅ Zusätzlich `Gemfile`/`Gemfile.lock` (Ruby-Toolchain) und ein `Tools/`-Ordner (JS-Generatoren für GitHub-Style/Prism).

> **Hinweis zum Ablageort.** Xcode-Projekt bewusst lokal unter `~/Programming/Xcode/` — **nicht** in der Nextcloud (`.xcodeproj`/Pods erzeugen sonst Sync-Konflikte). Da das Repo außerhalb der Nextcloud liegt, ist **normales Git** ok (kein `separate-git-dir`-Kniff nötig).

---

## 2. Änderung A — Suche im sichtbaren Vorschau-Text

### 2.1 Problem & Ursache (verifiziert)

- ✅ **MacDown hat keinen eigenen Suchcode.** Kein `NSTextFinder`/`performFindPanel…`/`findString`/`searchFor` in `MPDocument.m` oder `MPEditorView.m`. Die Suche ist vollständig die **Standard-macOS-Suche**.
- ✅ `MainMenu.xib:202–225` — alle Find-Menüpunkte hängen an `performFindPanelAction:` mit `target="-1"` (**First Responder**), Standard-Tags: `Find…` tag=1 (⌘F), `Find and Replace…` tag=12, `Find Next` tag=2 (⌘G), `Find Previous` tag=3 (⇧⌘G), `Use Selection for Find` tag=7 (⌘E).
- ✅ `MPEditorView.h:11` — `MPEditorView : NSTextView`. Ein `NSTextView` beantwortet `performFindPanelAction:` von Haus aus → **der Editor ist der einzige Responder in der Kette, der die Suche implementiert.**
- ✅ **Warum der versteckte Editor trotzdem gefunden wird:** „Editor ausblenden" **entfernt ihn nicht**, es kollabiert nur seine Breite auf 0 (siehe Abschnitt 4). `MPDocument.m:696` setzt beim Verstecken nur `self.editor.editable = self.editorVisible;` → der Editor bleibt **nicht-editierbar, aber in der View-Hierarchie und in der Responder-Chain**. Ein `NSTextView` sucht auch im nicht-editierbaren Zustand. `MPDocument.m:291`: `editorVisible = (self.editorContainer.frame.size.width != 0.0)`.
- ✅ Die `WebView`-Vorschau beantwortet `performFindPanelAction:` gar nicht → selbst wenn sie fokussiert wäre, liefe die Suche über die Responder-Chain zum Editor.

### 2.2 Lösung 🔶

Zwei Bausteine:

1. **Vorschau-Suche bauen** über die **öffentliche** WebView-Methode
   `- (BOOL)searchFor:(NSString*)string direction:(BOOL)forward caseSensitive:(BOOL)caseFlag wrap:(BOOL)wrapFlag;`
   Diese markiert die Fundstelle in der `WebView`. (Nicht `findString:withConfiguration:` — das ist `WKWebView`-only und hier nicht anwendbar.)
2. **⌘F sichtbarkeitsabhängig routen:** Wenn der Editor sichtbar ist → wie bisher die NSTextView-Find-Bar. Wenn der Editor versteckt ist (`!editorVisible`) → die eigene Vorschau-Suche.
   - 🔶 **Bevorzugter Weg:** `performFindPanelAction:` im `MPDocument` abfangen und je nach Sichtbarkeit an Editor-Find-Bar oder eigene WebView-Suche delegieren (ein zentraler Punkt, nutzt die vorhandene Menüstruktur, Shortcuts bleiben gleich).
   - Alternative: eigene Menüpunkte/Such-Toolbar nur für die Vorschau.

**Was der Löwenanteil der Arbeit ist:** `searchFor:` liefert nur „gefunden ja/nein", **keine fertige Find-Bar**. Wir müssen die kleine UI selbst bauen: Suchfeld, Weiter/Zurück, optional Trefferzähler (Trefferzahl ist mit `searchFor:` nicht direkt zu bekommen — ggf. per JS-Zählung über `mainFrame.javaScriptContext`, oder wir verzichten zunächst auf den Zähler).

### 2.3 Definition of Done (A)

- Editor sichtbar → ⌘F verhält sich **exakt wie heute** (Editor-Suche, Ersetzen funktioniert).
- Editor versteckt → ⌘F öffnet eine Suche, die im **gerenderten Vorschau-Text** sucht, die Fundstelle **sichtbar hervorhebt** und mit ⌘G / ⇧⌘G weiter/zurück springt.
- Kein Fokus mehr im unsichtbaren Editor beim Suchen im Reader-Modus.

**Aufwand:** klein–mittel (Kern klein, UI der Such-Leiste ist der Aufwand).

---

## 3. Änderung B — Zoom mit ⌘+ / ⌘−

### 3.1 Ist-Zustand (verifiziert)

- ✅ **Kein interaktiver Zoom, keine ⌘+/⌘−-Menüpunkte.** Der einzige „Zoom"-Eintrag im XIB (`MainMenu.xib:589`, ⌘M → `performZoom:`) ist das **Standard-Fenster-Zoom** (Window-Menü), nicht Inhalts-Zoom.
- ✅ Es existiert nur eine **preference-gekoppelte** Skalierung — `MPDocument.m:1691 scaleWebview`: `scale = fontSize/defaultSize; [self.preview setPageSizeMultiplier:scale];` gesteuert über `previewZoomRelativeToBaseFontSize` + `editorBaseFontSize`. Koppelt die Vorschaugröße an die **Editor**-Schriftgröße; **kein freier Live-Zoom**.
- ✅ **Stolperstein, im Code selbst notiert** (`MPDocument.m:1712`): *„this is private webkit API and NOT App Store-safe!"* — `setPageSizeMultiplier:` ist **private API** (`WebView+WebViewPrivateHeaders.h:18`). Für einen selbst-signierten Fork ok; für App-Store-Vertrieb ein Blocker.

### 3.2 Lösung 🔶

- **Sauberster Weg:** `WebView` hat die **öffentlichen** Actions `- zoomPageIn:` / `- zoomPageOut:` / `- resetPageZoom:` (plus `canZoomPageIn`/`canZoomPageOut` fürs Menü-Enabling). Zwei neue Menüpunkte **⌘+** und **⌘−** im „View"-Menü, `target="-1"`, die diese Actions auslösen — bzw. eigene IBActions im `MPDocument`, die den Zoomfaktor halten.
- **Menü-Ort:** In `MainMenu.xib` im „View"-Menü, bei den vorhandenen Ansicht-Einträgen (Bereich ~Zeile 380–401). Technisch ist ⌘+ der Key `=`, ⌘− der Key `-`.
- ⚠️ **Kollision prüfen:** `-` wird im XIB bereits als Shortcut für „Strikethrough/Durchstreichen" verwendet (`MainMenu.xib:450`, ggf. mit anderem Modifier-Kontext). Beim Vergeben von ⌘− prüfen, dass sich nichts überlagert.
- 🔶 **App-Store-saubere Alternative** (falls je relevant): CSS-Injection — Basis-`font-size` im Stylesheet hochsetzen und neu rendern. Mehr Aufwand, aber unabhängig von WebView-Interna.

### 3.3 Definition of Done (B)

- ⌘+ vergrößert, ⌘− verkleinert den **angezeigten Vorschau-Text** stufenweise; ein „Zurücksetzen" (⌘0) stellt die Standardgröße wieder her.
- Menüpunkte werden korrekt aktiviert/deaktiviert (`canZoomPageIn/Out`).
- Keine Kollision mit „Durchstreichen".

**Aufwand:** klein.

---

## 4. Änderung C — Editor als eigenes Fenster (OFFENE ENTSCHEIDUNG)

> **Status: noch nicht entschieden.** Das ist der große, riskante Brocken. Erst bauen, wenn der konkrete Nutzen geklärt ist (siehe 4.3).

### 4.1 Ist-Zustand (verifiziert)

- ✅ `MPDocument.m:187` — `IBOutlet MPDocumentSplitView *splitView;`
- ✅ `MPDocumentSplitView.m:24` — `MPDocumentSplitView : NSSplitView`, **strikt zwei Panes** (`NSAssert … parts.count == 2`). Links `editorContainer` (`MPDocument.m:188`), rechts `preview` (`WebView`).
- ✅ „Ausblenden" = **Divider auf 0.0 oder 1.0 fahren**, nicht Entfernen: `toggleEditorPane:` → `toggleSplitterCollapsingEditorPane:YES` (`:1481`) → Ziel-Ratio 0/1 (`:1499`, `:1511`) via `setSplitViewDividerLocation:`; `MPDocumentSplitView.setDividerLocation:` (`:54`) ruft `setPosition:ofDividerAtIndex:0`. Editor bleibt mit Breite 0 in der Hierarchie.
- ✅ Menü: `MainMenu.xib:392–401` — „Hide Preview Pane" (⇧⌘H → `togglePreviewPane:`) und „Hide Editor Pane" (⇧⌘E → `toggleEditorPane:`); Titel wechseln dynamisch in `validateUserInterfaceItem:` (`:667–685`).
- ✅ Es gibt bereits einen `swapViews`-Mechanismus (`MPDocumentSplitView.m:76`) für „Editor rechts/links" (`preferences.editorOnRight`).

### 4.2 Was ein eigenes Fenster berührt 🔶 (Aufwand mittel–groß, mit echten Risiken)

1. **Enge Kopplung Editor ↔ Renderer ↔ Vorschau.** Der Renderdurchlauf greift direkt auf `self.editor` (NSTextView) **und** `self.preview` (WebView) desselben `MPDocument` zu — z. B. Sync-Scrolling in `updateHeaderLocations` (ab `:1717`) und `syncScrollers` (ab `:1780`). Ein eigenes Fenster heißt: der Editor lebt in einem anderen `NSWindow`, aber Renderer und Vorschau erwarten ihn lokal. Diese Zugriffe müssten alle über eine Referenz statt „im selben Nib" laufen.
2. **Nib-Architektur.** Editor, SplitView und WebView sind zusammen in `MPDocument.xib` als IBOutlets verdrahtet. Ein separates Editorfenster braucht ein eigenes Nib + `NSWindowController` und eine explizite Rückverbindung zum Dokument.
3. **Responder-Chain & Menü-Validierung.** Viele IBActions (Markup wie `toggleStrong:` etc., ab `:1334`) gehen an First Responder und operieren auf `self.editor`. Sitzt der Editor in einem anderen Key-Window, ändert sich, welches `MPDocument` First Responder ist — Menü-Enabling und Markup-Aktionen müssen das berücksichtigen.
4. **Sichtbarkeitslogik.** `editorVisible`/`previewVisible`, `needsHtml` (`:294`, unterdrückt Rendern wenn Vorschau versteckt) und die Kollaps-Mechanik gehen von **einem** SplitView mit beiden Panes aus. Bei ausgelagertem Editor ist „previewVisible" immer true und die Kollaps-Logik entfällt/ändert sich.

**Risiko-Kernpunkte (was potenziell bricht):** Sync-Scrolling, Live-Rendering-Trigger über `needsHtml`, die Markup-Toolbar/Format-Aktionen über First Responder, sowie die Divider-/Swap-Logik. Machbar, aber **kein isolierter View-Umbau** — es greift in die Kernschleife des Dokuments ein.

### 4.3 Offene Frage (zu klären, bevor C gebaut wird)

Wofür genau ist die Trennung nötig?

- **(a) „Mir reicht der Reader":** Wenn Suche (A) + Zoom (B) im Reader-Modus laufen, ist C vermutlich **überflüssig** — dann bauen wir C nicht.
- **(b) „Erst A+B, dann C bewerten":** A+B zuerst ausprobieren, danach entscheiden. *(Empfehlung.)*
- **(c) „Zwei Fenster sind Pflicht":** Editor **und** Vorschau sollen **gleichzeitig in getrennten Fenstern** sichtbar sein. Nur dann lohnt der große Umbau — und dann planen wir ihn bewusst.

> Der vorhandene Kollaps-Mechanismus (Editor-Breite 0) leistet fast das, was ein „nur Vorschau"-Modus braucht. Ein echtes zweites Fenster lohnt sich **nur** für den Nutzen (c).

---

## 5. Empfohlene Reihenfolge

1. **Build zum Laufen bringen** (Abschnitt 7) — unverändertes MacDown erst einmal kompilieren und starten, damit wir eine saubere Basis haben.
2. **Änderung B (Zoom)** — kleinster, risikoärmster Gewinn; gute Aufwärmübung im Code.
3. **Änderung A (Suche)** — der eigentliche Haupt-Schmerz; etwas mehr UI-Arbeit.
4. **A+B ausprobieren lassen** (Udo testet live — „es kompiliert" ist nicht „fertig").
5. **Dann erst über C entscheiden** (Abschnitt 4.3).

**Warum diese Reihenfolge:** klein→groß, jeder Schritt für sich testbar, und C — der einzige teure/riskante Punkt — wird möglicherweise durch A+B überflüssig. So investieren wir den großen Aufwand erst, wenn er sich als nötig erwiesen hat.

---

## 6. Weggabelung für „später": `WebView` → `WKWebView`

**Heute fast gratis festzuhalten, teuer wenn übersehen:** Die gesamte Vorschau steht auf der deprecated `WebView`. Solange A/B klein bleiben, ist das ok. Aber A/B setzen bewusst auf **WebView-Spezifika** auf (`searchFor:…`, `zoomPageIn:`, `setPageSizeMultiplier:`). Sobald eines dieser Ziele dazukommt —

- eine „richtige", moderne Find-in-Page-Oberfläche,
- moderner/robuster Zoom,
- **App-Store-Vertrieb** (private API `setPageSizeMultiplier:` ist dort ein Blocker),

— führt kein Weg an einem Umstieg auf `WKWebView` vorbei. Das ist ein **eigenes großes Projekt**: anderes Delegate-Modell (`WKNavigationDelegate` statt `WebFrameLoadDelegate`), **kein** `mainFrame.javaScriptContext` (JS läuft asynchron über `evaluateJavaScript:`), und die synchronen Sync-Scroll-/Header-Berechnungen müssten auf async umgestellt werden.

**Handlungsempfehlung heute:** A/B so kapseln, dass der WebView-Zugriff an **wenigen, klar benannten Stellen** sitzt (z. B. eine kleine Kategorie/Hilfsklasse „PreviewSearch"/„PreviewZoom"), damit ein späterer WKWebView-Umstieg diese Punkte gezielt ersetzen kann, statt WebView-Aufrufe über den ganzen Code zu verstreuen.

---

## 7. Setup-Schritte zum Loslegen (wenn wir bauen)

Kurze Einzelschritte, jeder einzeln prüfbar (keine langen Skripte):

1. `cd ~/Programming/Xcode/macdown`
2. `git submodule update --init` (holt `Dependency/prism`).
3. CocoaPods bereitstellen (idealerweise Version nahe **1.8.4**; via Bundler `bundle install` + `bundle exec pod install`, oder system-`pod install`). Achtung auf die gepatchten Specs von `MacDownApp/cocoapods-specs`.
4. `MacDown.xcworkspace` in Xcode öffnen (**nicht** das `.xcodeproj`).
5. Signing auf Udos bezahltes Team stellen (Team-ID `TFFAN6MCLF`, Automatic) — Vorsicht: Zertifikatsnamen zeigen teils das persönliche Team `8P563RGCJ6` an, nicht das bezahlte.
6. Unverändert bauen & starten → **Baseline bestätigen**, bevor irgendetwas geändert wird.
7. Erst danach: Branch anlegen (`feature/reader-search-zoom`), Änderung B, dann A.

**Fork/Veröffentlichung (später, nicht jetzt):** GitHub-Fork von `MacDownApp/macdown` für die öffentliche Fork-Version; zusätzlich Spiegelung in Udos eigenes Gitea (`git.udo-schmitz.de`, SSH-Key, Port 2222) nach der üblichen Konvention. Default-Branch `main`.

---

## 8. Offene Entscheidungen (Checkliste)

- [x] **C-Entscheidung** (Abschnitt 4.3): **(b) erst A+B, dann C bewerten** — entschieden von Udo am 2026-10-18. Die eigentliche Frage „brauche ich überhaupt ein zweites Fenster?“ wird nach dem Live-Test von A+B gestellt.
- [ ] **Umgang mit der installierten Fremd-Build** in `/Applications` (Mass Metal, 1.0.0): durch die eigene ersetzen, oder daneben behalten? Gleiche Bundle-ID → 🔶 vermutlich teilen sich beide die Einstellungen (`~/Library/Preferences/com.uranusjr.macdown.plist`, falls die Fremd-Build nicht sandboxed ist) und konkurrieren um die Dateizuordnung für `.md`. **Noch nicht geprüft** — vor der Entscheidung mit `codesign -d --entitlements :- /Applications/MacDown.app` und einem Blick in `~/Library/Containers/` verifizieren. Spätestens vor dem ersten Live-Test klären.
- [ ] **Trefferzähler in der Vorschau-Suche** (A): mit JS-Zählung, oder zunächst weglassen?
- [ ] **⌘−-Kollision** mit „Durchstreichen" (B) endgültig prüfen und ggf. den Strikethrough-Shortcut anpassen.
- [ ] **App-Store-Ambition ja/nein** — entscheidet, ob wir bei privater API (`setPageSizeMultiplier:`) bleiben dürfen oder gleich sauber (öffentliche Zoom-Actions / CSS) bauen und den WKWebView-Umstieg (Abschnitt 6) einplanen.

---

## Anhang: Verifizierte Datei:Zeile-Referenzen (Schnellzugriff beim Bauen)

| Thema | Fundstelle |
|---|---|
| Vorschau = `WebView` | `MPDocument.m:10`, `:176`, `:191`, `:1722` |
| Find-Menü → First Responder | `MainMenu.xib:202–225` |
| Editor ist `NSTextView` | `MPEditorView.h:11` |
| Editor bleibt beim Verstecken aktiv-in-Kette | `MPDocument.m:291`, `:696` |
| Preview-Skalierung (private API!) | `MPDocument.m:1691`, `:1712`; `WebView+WebViewPrivateHeaders.h:18` |
| Fenster-Zoom (nicht Inhalt) | `MainMenu.xib:589` |
| SplitView, 2 Panes | `MPDocument.m:187`; `MPDocumentSplitView.m:24`, `:46/57` |
| Pane-Kollaps / Divider | `MPDocument.m:1481`, `:1499`, `:1511`; `MPDocumentSplitView.m:54` |
| Hide-Menüpunkte | `MainMenu.xib:392–401`; Validierung `MPDocument.m:667–685` |
| Editor-Swap L/R | `MPDocumentSplitView.m:76` |
| Sync-Scroll (bricht bei C potenziell) | `MPDocument.m:1717`, `:1780` |
| Markup-Aktionen an First Responder | `MPDocument.m:1334` ff. |
| Rendern nur wenn Vorschau sichtbar | `MPDocument.m:294` |

*Legende: ✅ im Code verifiziert · 🔶 Design-Empfehlung/Einschätzung. „Es kompiliert" ist nicht „fertig" — UI-Änderungen werden live von Udo geprüft, bevor committet wird.*

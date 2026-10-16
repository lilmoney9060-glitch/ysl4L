# ysl4L
my first open sourcecode

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="theme-color" content="#0f172a">
  <title>ClearDuplicate | File Scanner</title>

  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      padding: 16px;
      background: #0f172a;
      color: #f8fafc;
      font-family: system-ui, sans-serif;
    }

    .card {
      max-width: 750px;
      margin: 0 auto;
      padding: 20px;
      background: #1e293b;
      border: 1px solid #334155;
      border-radius: 16px;
    }

    h1 {
      margin-top: 0;
      font-size: 27px;
    }

    .muted {
      color: #94a3b8;
      line-height: 1.6;
      font-size: 14px;
    }

    input[type="file"] {
      width: 100%;
      margin: 12px 0;
      padding: 12px;
      background: #334155;
      color: white;
      border-radius: 8px;
      font-size: 13px;
    }

    button {
      width: 100%;
      margin-top: 10px;
      padding: 14px;
      border: 0;
      border-radius: 8px;
      font-weight: 700;
      font-size: 15px;
      cursor: pointer;
    }

    button:disabled {
      opacity: 0.55;
      cursor: wait;
    }

    #scanBtn {
      background: #3b82f6;
      color: white;
    }

    #reportBtn {
      background: #22c55e;
      color: #052e16;
    }

    #results {
      margin-top: 20px;
    }

    .summary {
      padding: 14px;
      background: #0f172a;
      border-radius: 10px;
      line-height: 1.8;
      overflow-wrap: anywhere;
    }

    .group {
      margin-top: 18px;
      padding: 12px;
      background: #0f172a;
      border-radius: 10px;
    }

    .group h3 {
      margin-top: 0;
      font-size: 15px;
      overflow-wrap: anywhere;
    }

    .item {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      gap: 12px;
      padding: 12px 8px;
      margin-top: 8px;
      border-radius: 8px;
      background: #334155;
      border-left: 4px solid #ef4444;
    }

    .item.keep {
      border-left-color: #22c55e;
    }

    .filename {
      overflow-wrap: anywhere;
      font-size: 14px;
    }

    .badge {
      display: inline-block;
      padding: 4px 7px;
      margin-bottom: 6px;
      border-radius: 5px;
      font-size: 11px;
      font-weight: 700;
    }

    .green {
      background: #22c55e;
      color: #052e16;
    }

    .red {
      background: #7f1d1d;
      color: #fecaca;
    }

    .details {
      flex-shrink: 0;
      text-align: right;
      color: #cbd5e1;
      font-size: 12px;
    }

    .success {
      color: #4ade80;
    }

    .warning {
      color: #fbbf24;
    }

    .notice {
      margin-top: 16px;
      padding: 12px;
      background: #422006;
      color: #fde68a;
      border-radius: 8px;
      font-size: 13px;
      line-height: 1.6;
    }

    #reportBtn {
      display: none;
    }
  </style>
</head>

<body>
  <main class="card">
    <h1>🧹 ClearDuplicate</h1>

    <p class="muted">
      Find exact duplicate files using SHA-256 content
      fingerprints. Your selected files are processed locally
      in your browser.
    </p>

    <label for="files">Choose files to scan</label>
    <input
      type="file"
      id="files"
      multiple
      accept="image/*,audio/*,video/*,.pdf,.doc,.docx,.txt,.json,.csv"
    >

    <button id="scanBtn">🔍 Scan for Duplicates</button>

    <div id="status" class="muted" role="status" aria-live="polite">
      Select files to begin.
    </div>

    <div id="results"></div>

    <button id="reportBtn">⬇ Download Scan Report</button>

    <div class="notice">
      Important: This website cannot automatically access or
      delete files from your phone. It scans only files you
      select. Review the report before deleting anything.
    </div>
  </main>

  <script>
    "use strict";

    const fileInput = document.getElementById("files");
    const scanBtn = document.getElementById("scanBtn");
    const reportBtn = document.getElementById("reportBtn");
    const statusBox = document.getElementById("status");
    const results = document.getElementById("results");

    let lastReport = "";
    let scanVersion = 0;

    function formatBytes(bytes) {
      if (bytes === 0) return "0 B";

      const units = ["B", "KB", "MB", "GB"];
      const index = Math.min(
        Math.floor(Math.log(bytes) / Math.log(1024)),
        units.length - 1
      );

      return (
        (bytes / Math.pow(1024, index)).toFixed(
          index === 0 ? 0 : 2
        ) + " " + units[index]
      );
    }

    function makeElement(tag, className, text) {
      const element = document.createElement(tag);

      if (className) element.className = className;
      if (text !== undefined) element.textContent = text;

      return element;
    }

    async function getDimensions(file) {
      if (!file.type.startsWith("image/")) {
        return "";
      }

      if (typeof createImageBitmap !== "function") {
        return "Image dimensions unavailable";
      }

      try {
        const bitmap = await createImageBitmap(file);

        const dimensions =
          bitmap.width + " × " + bitmap.height + " px";

        bitmap.close();
        return dimensions;
      } catch {
        return "Image dimensions unavailable";
      }
    }

    async function getHash(file) {
      const buffer = await file.arrayBuffer();

      const digest = await crypto.subtle.digest(
        "SHA-256",
        buffer
      );

      return Array.from(new Uint8Array(digest))
        .map(byte => byte.toString(16).padStart(2, "0"))
        .join("");
    }

    function createFileRow(file, isKeep, dimensions) {
      const row = makeElement(
        "div",
        isKeep ? "item keep" : "item"
      );

      const info = makeElement("div");
      const badge = makeElement(
        "span",
        "badge " + (isKeep ? "green" : "red"),
        isKeep ? "KEEP ONE COPY" : "DUPLICATE"
      );

      info.appendChild(badge);
      info.appendChild(
        makeElement("div", "filename", file.name)
      );

      const detailText = [
        formatBytes(file.size),
        dimensions || "Non-image file",
        file.type || "Unknown file type"
      ].join("\n");

      const details = makeElement("div", "details");
      details.style.whiteSpace = "pre-line";
      details.textContent = detailText;

      row.appendChild(info);
      row.appendChild(details);

      return row;
    }

    function renderGroup(group, number) {
      const section = makeElement("section", "group");

      section.appendChild(
        makeElement(
          "h3",
          "",
          "Duplicate group " + number +
          " (" + group.length + " files)"
        )
      );

      group.forEach((item, index) => {
        section.appendChild(
          createFileRow(
            item.file,
            index === 0,
            item.dimensions
          )
        );
      });

      return section;
    }

    scanBtn.addEventListener("click", async () => {
      const files = Array.from(fileInput.files);

      if (files.length === 0) {
        alert("Please select files first.");
        return;
      }

      if (!globalThis.crypto?.subtle) {
        statusBox.textContent =
          "Secure hashing is unavailable. Open this page over HTTPS.";
        return;
      }

      const currentScan = ++scanVersion;

      scanBtn.disabled = true;
      reportBtn.style.display = "none";
      results.replaceChildren();
      lastReport = "";

      try {
        statusBox.textContent =
          "Preparing to scan " + files.length + " files...";

        const groups = new Map();

        for (let i = 0; i < files.length; i++) {
          if (currentScan !== scanVersion) return;

          const file = files[i];

          statusBox.textContent =
            "Checking file " + (i + 1) + " of " +
            files.length + ": " + file.name;

          const hash = await getHash(file);

          if (!groups.has(hash)) {
            groups.set(hash, []);
          }

          groups.get(hash).push({
            file,
            dimensions: await getDimensions(file),
            hash
          });
        }

        if (currentScan !== scanVersion) return;

        const duplicates = [];
        let duplicateCopies = 0;
        let wastedBytes = 0;

        for (const [hash, group] of groups) {
          if (group.length < 2) continue;

          // All members have the same content hash.
          // Keep the first selected file as the reference copy.
          duplicates.push({ hash, group });

          duplicateCopies += group.length - 1;

          for (let i = 1; i < group.length; i++) {
            wastedBytes += group[i].file.size;
          }
        }

        const summary = makeElement("div", "summary");

        summary.appendChild(
          makeElement("div", "", "Files scanned: " + files.length)
        );

        summary.appendChild(
          makeElement(
            "div",
            "",
            "Duplicate groups: " + duplicates.length
          )
        );

        summary.appendChild(
          makeElement(
            "div",
            "",
            "Extra copies: " + duplicateCopies
          )
        );

        summary.appendChild(
          makeElement(
            "div",
            "",
            "Potential duplicate storage: " +
              formatBytes(wastedBytes)
          )
        );

        results.appendChild(summary);

        const reportLines = [
          "ClearDuplicate Scan Report",
          "Date: " + new Date().toISOString(),
          "Files scanned: " + files.length,
          "Duplicate groups: " + duplicates.length,
          "Extra copies: " + duplicateCopies,
          "Potential duplicate storage: " +
            formatBytes(wastedBytes),
          "",
          "Note: No files were deleted."
        ];

        if (duplicates.length === 0) {
          results.appendChild(
            makeElement(
              "p",
              "success",
              "No exact duplicate files were found."
            )
          );

          reportLines.push("", "No exact duplicates found.");
        } else {
          duplicates.forEach(({ hash, group }, index) => {
            results.appendChild(renderGroup(group, index + 1));

            reportLines.push(
              "",
              "Group " + (index + 1),
              "SHA-256: " + hash
            );

            group.forEach((item, itemIndex) => {
              reportLines.push(
                (itemIndex === 0 ? "KEEP: " : "DUPLICATE: ") +
                item.file.name +
                " | " + formatBytes(item.file.size) +
                " | " + (item.dimensions || "Non-image file")
              );
            });
          });
        }

        lastReport = reportLines.join("\n");
        reportBtn.style.display = "block";

        statusBox.textContent = "Scan completed successfully.";
        statusBox.className = "muted success";

      } catch (error) {
        console.error(error);

        statusBox.textContent =
          "Scan failed. Try fewer or smaller files, then scan again.";

        statusBox.className = "muted warning";

      } finally {
        scanBtn.disabled = false;
      }
    });

    reportBtn.addEventListener("click", () => {
      if (!lastReport) return;

      const blob = new Blob(
        [lastReport],
        { type: "text/plain;charset=utf-8" }
      );

      const url = URL.createObjectURL(blob);
      const link = document.createElement("a");

      link.href = url;
      link.download = "clearduplicate-report.txt";

      document.body.appendChild(link);
      link.click();
      link.remove();

      setTimeout(() => URL.revokeObjectURL(url), 1000);
    });

    fileInput.addEventListener("change", () => {
      scanVersion++;
      lastReport = "";
      results.replaceChildren();
      reportBtn.style.display = "none";
      statusBox.className = "muted";
      statusBox.textContent =
        fileInput.files.length + " files selected.";
    });
  </script>
</body>
</html>

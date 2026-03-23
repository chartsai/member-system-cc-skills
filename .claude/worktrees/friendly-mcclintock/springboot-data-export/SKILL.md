---
name: springboot-data-export
description: >
  TRIGGER this skill when the user says "add data export", "export members to CSV", "export to Excel",
  "run springboot-data-export", "add CSV export", "add Excel export", "export data to Google Sheets",
  "admin should be able to download member data", or anything about exporting database records to files.
  Requires setup, scaffold, db, membership. Google Sheets export additionally needs gcp module.
---

# springboot-data-export

Adds admin data export: CSV, Excel (XLSX), and optionally Google Sheets (if `gcp` module installed).
Configurable field mapping, value translation, and table whitelist for security.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `membership` must be installed

### Soft prerequisites
- `gcp` → enables Google Sheets export (gracefully disabled if absent)

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
app_name          → used in export file names
gcp_project_id    → needed for Google Sheets export
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → must contain "setup", "scaffold", "db", "membership"
                    "gcp" → enables Google Sheets export
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

If `"gcp"` is missing, note:
```
Note: gcp module is not installed. Google Sheets export will not be available.
CSV and Excel export will still work. Install springboot-gcp-setup first to enable Google Sheets.
```

---

## Step 2 — Update `build.gradle.kts`

Add:

```kotlin
// Excel export
implementation("org.apache.poi:poi-ooxml:5.3.0")

// CSV export
implementation("com.opencsv:opencsv:5.9")

// Google Sheets API (only if gcp module is installed)
// Add only if "gcp" is in installed_modules:
implementation("com.google.apis:google-api-services-sheets:v4-rev20220927-2.0.0")
implementation("com.google.auth:google-auth-library-oauth2-http:1.23.0")
```

---

## Step 3 — Export configuration

Create `src/main/resources/export-config.yml` (or use `app_config` table if `app-config` module is installed):

```yaml
exports:
  members:
    table: members
    fields:
      - column: id
        label: "ID"
      - column: email
        label: "Email"
      - column: name
        label: "Full Name"
      - column: role
        label: "Role"
        valueMap:
          MEMBER: "Member"
          ADMIN: "Admin"
          SUPER_ADMIN: "Super Admin"
      - column: status
        label: "Status"
        valueMap:
          ACTIVE: "Active"
          PENDING: "Pending"
          SUSPENDED: "Suspended"
      - column: created_at
        label: "Joined At"
        format: "yyyy-MM-dd"
```

---

## Step 4 — `ExportFieldConfig` — configuration model

```java
package {{base_package}}.export;

import lombok.*;
import java.util.List;
import java.util.Map;

@Getter @Setter @NoArgsConstructor
public class ExportFieldConfig {
    private String column;
    private String label;
    private Map<String, String> valueMap;  // Optional: coded value → display value
    private String format;                  // Optional: date format
}

// And a container:
@Getter @Setter @NoArgsConstructor
public class ExportTableConfig {
    private String table;                   // DB table name (whitelist-controlled)
    private List<ExportFieldConfig> fields;
}
```

---

## Step 5 — Table whitelist (security)

**Critical security note:** Never allow arbitrary table names from user input. Always whitelist.

```java
package {{base_package}}.export;

import java.util.Set;

public final class ExportTableWhitelist {
    /** Only these table names may be exported via the export API. */
    public static final Set<String> ALLOWED_TABLES = Set.of(
        "members",
        "applications",
        "submissions"
        // Add other exportable tables here explicitly
    );

    public static void validate(String tableName) {
        if (!ALLOWED_TABLES.contains(tableName)) {
            throw new SecurityException("Export not allowed for table: " + tableName);
        }
    }

    private ExportTableWhitelist() {}
}
```

---

## Step 6 — `DataExportService`

```java
package {{base_package}}.service;

import {{base_package}}.export.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import java.io.*;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.*;

@Service
@Slf4j
@RequiredArgsConstructor
public class DataExportService {

    private final JdbcTemplate jdbcTemplate;
    private final ExportConfigLoader configLoader;  // loads from export-config.yml

    /**
     * Export a table to CSV bytes.
     */
    public byte[] exportToCsv(String tableName) throws IOException {
        ExportTableWhitelist.validate(tableName);
        ExportTableConfig config = configLoader.getConfig(tableName);
        List<Map<String, Object>> rows = fetchRows(tableName, config);

        ByteArrayOutputStream out = new ByteArrayOutputStream();
        try (var writer = new java.io.OutputStreamWriter(out);
             var csvWriter = new com.opencsv.CSVWriter(writer)) {

            // Header
            String[] header = config.getFields().stream()
                .map(ExportFieldConfig::getLabel)
                .toArray(String[]::new);
            csvWriter.writeNext(header);

            // Rows
            for (Map<String, Object> row : rows) {
                String[] values = config.getFields().stream()
                    .map(field -> formatValue(row.get(field.getColumn()), field))
                    .toArray(String[]::new);
                csvWriter.writeNext(values);
            }
        }
        return out.toByteArray();
    }

    /**
     * Export a table to XLSX bytes.
     */
    public byte[] exportToXlsx(String tableName) throws IOException {
        ExportTableWhitelist.validate(tableName);
        ExportTableConfig config = configLoader.getConfig(tableName);
        List<Map<String, Object>> rows = fetchRows(tableName, config);

        try (Workbook workbook = new XSSFWorkbook()) {
            Sheet sheet = workbook.createSheet(tableName);

            // Header row
            Row headerRow = sheet.createRow(0);
            CellStyle headerStyle = workbook.createCellStyle();
            Font font = workbook.createFont();
            font.setBold(true);
            headerStyle.setFont(font);

            List<ExportFieldConfig> fields = config.getFields();
            for (int i = 0; i < fields.size(); i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue(fields.get(i).getLabel());
                cell.setCellStyle(headerStyle);
            }

            // Data rows
            int rowNum = 1;
            for (Map<String, Object> row : rows) {
                Row dataRow = sheet.createRow(rowNum++);
                for (int i = 0; i < fields.size(); i++) {
                    Object value = row.get(fields.get(i).getColumn());
                    dataRow.createCell(i).setCellValue(formatValue(value, fields.get(i)));
                }
            }

            // Auto-size columns
            for (int i = 0; i < fields.size(); i++) {
                sheet.autoSizeColumn(i);
            }

            ByteArrayOutputStream out = new ByteArrayOutputStream();
            workbook.write(out);
            return out.toByteArray();
        }
    }

    private List<Map<String, Object>> fetchRows(String tableName, ExportTableConfig config) {
        ExportTableWhitelist.validate(tableName);
        String columns = config.getFields().stream()
            .map(ExportFieldConfig::getColumn)
            .reduce((a, b) -> a + ", " + b)
            .orElse("*");
        // Safe: tableName is whitelisted above; columns come from config, not user input
        return jdbcTemplate.queryForList("SELECT " + columns + " FROM " + tableName);
    }

    private String formatValue(Object value, ExportFieldConfig field) {
        if (value == null) return "";

        // Apply valueMap translation if defined
        if (field.getValueMap() != null) {
            String mapped = field.getValueMap().get(value.toString());
            if (mapped != null) return mapped;
        }

        // Apply date format if defined
        if (field.getFormat() != null && value instanceof java.sql.Timestamp ts) {
            return DateTimeFormatter.ofPattern(field.getFormat())
                .format(ts.toLocalDateTime());
        }

        return value.toString();
    }
}
```

---

## Step 7 — `AdminExportController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.service.DataExportService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.*;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import java.time.LocalDate;

@Controller
@RequestMapping("/admin/export")
@PreAuthorize("hasAnyRole('ADMIN', 'SUPER_ADMIN')")
@RequiredArgsConstructor
public class AdminExportController {

    private final DataExportService dataExportService;

    @GetMapping
    public String exportPage(Model model) {
        model.addAttribute("allowedTables",
            {{base_package}}.export.ExportTableWhitelist.ALLOWED_TABLES);
        return "admin/export/index";
    }

    @GetMapping("/csv/{table}")
    public ResponseEntity<byte[]> downloadCsv(@PathVariable String table) throws Exception {
        byte[] data = dataExportService.exportToCsv(table);
        String filename = table + "-" + LocalDate.now() + ".csv";
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(data);
    }

    @GetMapping("/xlsx/{table}")
    public ResponseEntity<byte[]> downloadXlsx(@PathVariable String table) throws Exception {
        byte[] data = dataExportService.exportToXlsx(table);
        String filename = table + "-" + LocalDate.now() + ".xlsx";
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .contentType(MediaType.parseMediaType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .body(data);
    }
}
```

---

## Step 8 — Google Sheets export (only if `gcp` installed)

If `"gcp"` is in `installed_modules`, create `GoogleSheetsExportService`:

```java
package {{base_package}}.service;

import com.google.api.services.sheets.v4.Sheets;
import com.google.api.services.sheets.v4.model.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import java.util.*;

@Service
@Profile("prod")   // Only available in production where GCP credentials are present
@Slf4j
@RequiredArgsConstructor
public class GoogleSheetsExportService {

    private final Sheets sheetsService;
    private final DataExportService dataExportService;

    /**
     * Create a new Google Sheet with the exported data and return its URL.
     */
    public String exportToGoogleSheets(String tableName, String sheetTitle) throws Exception {
        // Create a new spreadsheet
        Spreadsheet spreadsheet = new Spreadsheet()
            .setProperties(new SpreadsheetProperties().setTitle(sheetTitle));
        Spreadsheet created = sheetsService.spreadsheets().create(spreadsheet).execute();
        String spreadsheetId = created.getSpreadsheetId();

        // Get data as list of rows
        List<List<Object>> values = dataExportService.getRowsForSheets(tableName);

        // Write to sheet
        ValueRange body = new ValueRange().setValues(values);
        sheetsService.spreadsheets().values()
            .update(spreadsheetId, "A1", body)
            .setValueInputOption("RAW")
            .execute();

        String url = "https://docs.google.com/spreadsheets/d/" + spreadsheetId;
        log.info("Exported {} to Google Sheets: {}", tableName, url);
        return url;
    }
}
```

Wire up a `Sheets` bean in a `GcpConfig` configuration class using service account credentials.

Add a new endpoint to `AdminExportController`:

```java
@PostMapping("/sheets/{table}")
@Profile("prod")
public String exportToSheets(@PathVariable String table,
                              RedirectAttributes redirectAttributes) {
    try {
        String sheetTitle = table + " Export " + LocalDate.now();
        String url = googleSheetsExportService.exportToGoogleSheets(table, sheetTitle);
        redirectAttributes.addFlashAttribute("sheetsUrl", url);
        redirectAttributes.addFlashAttribute("success", "Exported to Google Sheets successfully.");
    } catch (Exception e) {
        redirectAttributes.addFlashAttribute("error", "Google Sheets export failed: " + e.getMessage());
    }
    return "redirect:/admin/export";
}
```

---

## Step 9 — `templates/admin/export/index.html`

Export page:
- Heading: "Data Export"
- Table of exportable data sets: Members, Applications, Submissions
- For each: Download CSV button, Download Excel button
- If GCP + prod profile: Google Sheets export button (POST form)
- Note about Google Sheets: "Available in production only"

---

## Step 10 — Unit tests for field mapping and value translation

If `test_mode` is NOT `"token-save"`:

```java
package {{base_package}}.export;

// Test: formatValue with valueMap: "ACTIVE" → "Active"
// Test: formatValue without valueMap → returns toString
// Test: formatValue with null → returns ""
// Test: ExportTableWhitelist.validate("members") → passes
// Test: ExportTableWhitelist.validate("users") → throws SecurityException
```

---

## Step 11 — Verify

Apply `test_mode`.

---

## Step 12 — Git commits

```bash
git add -A
git commit -m "feat: add admin data export system (CSV, Excel, optional Google Sheets)"
```

---

## Step 13 — Update `installed_modules`

Add `"data-export"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Data export is ready.

Admin export page: http://localhost:8080/admin/export
- CSV export: downloads immediately
- Excel export: downloads immediately
- Google Sheets: available in prod profile only (requires gcp module)

Security: Only tables in ExportTableWhitelist.ALLOWED_TABLES can be exported.
To add a new exportable table, add it to the whitelist AND to export-config.yml.
```

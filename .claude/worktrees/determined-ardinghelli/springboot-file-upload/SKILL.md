---
name: springboot-file-upload
description: >
  TRIGGER this skill when the user says "add file upload", "add file storage", "add attachment support",
  "run springboot-file-upload", "let members upload files", "add document upload", "store files in GCS",
  "add profile picture upload", or anything about allowing users to upload files that are stored locally
  or in Google Cloud Storage. Requires setup, scaffold, db. GCS storage additionally needs gcp module.
---

# springboot-file-upload

Adds a generic file upload system with a `FileStorageService` interface, local dev storage,
and optional GCS production storage. Files are tracked in the `uploaded_files` table.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
gcp_project_id    → needed if gcp module installed
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → must contain "setup", "scaffold", "db"
                    "gcp" → enables GcsFileStorageService
```

Note if `"gcp"` is in `installed_modules` — GCS service will be wired up.

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

---

## Step 2 — Update `build.gradle.kts`

Add:

```kotlin
// File upload
implementation("org.springframework.boot:spring-boot-starter-web")  // already present

// GCS (only if gcp module installed)
// Add only if "gcp" is in installed_modules:
implementation("com.google.cloud:spring-cloud-gcp-starter-storage:5.5.0")
```

If GCP module is NOT installed, skip the GCS dependency.

---

## Step 3 — Flyway migration

Create `V_upload__create_uploaded_files.sql` (use next available version number):

```sql
CREATE TABLE uploaded_files (
    id                BIGSERIAL PRIMARY KEY,
    member_id         BIGINT       REFERENCES members(id) ON DELETE SET NULL,
    filename          VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    content_type      VARCHAR(100),
    size_bytes        BIGINT       NOT NULL,
    storage_path      VARCHAR(1024) NOT NULL,
    storage_type      VARCHAR(20)  NOT NULL DEFAULT 'LOCAL'
                      CHECK (storage_type IN ('LOCAL', 'GCS')),
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_uploaded_files_member_id ON uploaded_files (member_id);
```

---

## Step 4 — `UploadedFile` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "uploaded_files")
@Getter @Setter @NoArgsConstructor @Builder @AllArgsConstructor
public class UploadedFile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @Column(nullable = false)
    private String filename;

    @Column(name = "original_filename", nullable = false)
    private String originalFilename;

    @Column(name = "content_type")
    private String contentType;

    @Column(name = "size_bytes", nullable = false)
    private Long sizeBytes;

    @Column(name = "storage_path", nullable = false)
    private String storagePath;

    @Column(name = "storage_type", nullable = false)
    @Enumerated(EnumType.STRING)
    private StorageType storageType = StorageType.LOCAL;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @PrePersist
    void onCreate() { createdAt = OffsetDateTime.now(); }

    public enum StorageType { LOCAL, GCS }
}
```

---

## Step 5 — `FileStorageService` interface

```java
package {{base_package}}.storage;

import org.springframework.web.multipart.MultipartFile;
import java.io.InputStream;

/**
 * Abstraction over file storage backends.
 * Local implementation stores files on disk (dev).
 * GCS implementation stores files in Google Cloud Storage (prod).
 */
public interface FileStorageService {

    /**
     * Store a file and return the storage path (relative or GCS object path).
     */
    String store(MultipartFile file, String directory);

    /**
     * Retrieve a file as InputStream.
     */
    InputStream retrieve(String storagePath);

    /**
     * Delete a file.
     */
    void delete(String storagePath);

    /**
     * Get a publicly accessible URL (or null if not applicable for this storage type).
     */
    String getPublicUrl(String storagePath);
}
```

---

## Step 6 — `LocalFileStorageService`

```java
package {{base_package}}.storage;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import java.io.*;
import java.nio.file.*;
import java.util.UUID;

@Service
@Profile("local")
@Slf4j
public class LocalFileStorageService implements FileStorageService {

    @Value("${app.storage.local.base-dir:./uploads}")
    private String baseDir;

    @Override
    public String store(MultipartFile file, String directory) {
        try {
            Path dir = Paths.get(baseDir, directory);
            Files.createDirectories(dir);

            String extension = getExtension(file.getOriginalFilename());
            String filename  = UUID.randomUUID() + extension;
            Path target = dir.resolve(filename);
            Files.copy(file.getInputStream(), target, StandardCopyOption.REPLACE_EXISTING);

            String storagePath = directory + "/" + filename;
            log.debug("Stored file locally: {}", storagePath);
            return storagePath;

        } catch (IOException e) {
            throw new RuntimeException("Failed to store file locally", e);
        }
    }

    @Override
    public InputStream retrieve(String storagePath) {
        try {
            return new FileInputStream(Paths.get(baseDir, storagePath).toFile());
        } catch (FileNotFoundException e) {
            throw new RuntimeException("File not found: " + storagePath, e);
        }
    }

    @Override
    public void delete(String storagePath) {
        try {
            Files.deleteIfExists(Paths.get(baseDir, storagePath));
        } catch (IOException e) {
            log.warn("Could not delete file: {}", storagePath, e);
        }
    }

    @Override
    public String getPublicUrl(String storagePath) {
        // Local files are served via the /files/** endpoint
        return "/files/download?path=" + java.net.URLEncoder.encode(storagePath, java.nio.charset.StandardCharsets.UTF_8);
    }

    private String getExtension(String filename) {
        if (filename == null || !filename.contains(".")) return "";
        return filename.substring(filename.lastIndexOf("."));
    }
}
```

---

## Step 7 — `GcsFileStorageService` (only if gcp module installed)

Create only if `"gcp"` is in `installed_modules`:

```java
package {{base_package}}.storage;

import com.google.cloud.storage.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import java.io.*;
import java.util.UUID;

@Service
@Profile("prod")
@Slf4j
public class GcsFileStorageService implements FileStorageService {

    private final Storage gcsStorage;

    @Value("${app.storage.gcs.bucket-name}")
    private String bucketName;

    public GcsFileStorageService(Storage gcsStorage) {
        this.gcsStorage = gcsStorage;
    }

    @Override
    public String store(MultipartFile file, String directory) {
        try {
            String extension = getExtension(file.getOriginalFilename());
            String objectName = directory + "/" + UUID.randomUUID() + extension;

            BlobId blobId = BlobId.of(bucketName, objectName);
            BlobInfo blobInfo = BlobInfo.newBuilder(blobId)
                .setContentType(file.getContentType())
                .build();

            gcsStorage.create(blobInfo, file.getBytes());
            log.debug("Stored file in GCS: {}/{}", bucketName, objectName);
            return objectName;

        } catch (IOException e) {
            throw new RuntimeException("Failed to store file in GCS", e);
        }
    }

    @Override
    public InputStream retrieve(String storagePath) {
        Blob blob = gcsStorage.get(BlobId.of(bucketName, storagePath));
        if (blob == null) throw new RuntimeException("File not found in GCS: " + storagePath);
        return new ByteArrayInputStream(blob.getContent());
    }

    @Override
    public void delete(String storagePath) {
        gcsStorage.delete(BlobId.of(bucketName, storagePath));
    }

    @Override
    public String getPublicUrl(String storagePath) {
        return "https://storage.googleapis.com/" + bucketName + "/" + storagePath;
    }

    private String getExtension(String filename) {
        if (filename == null || !filename.contains(".")) return "";
        return filename.substring(filename.lastIndexOf("."));
    }
}
```

---

## Step 8 — `FileUploadService` — orchestrates validation + storage + DB

```java
package {{base_package}}.service;

import {{base_package}}.entity.Member;
import {{base_package}}.entity.UploadedFile;
import {{base_package}}.repository.MemberRepository;
import {{base_package}}.repository.UploadedFileRepository;
import {{base_package}}.storage.FileStorageService;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;
import java.util.List;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class FileUploadService {

    private final FileStorageService fileStorageService;
    private final UploadedFileRepository uploadedFileRepository;
    private final MemberRepository memberRepository;

    @Value("${app.storage.max-size-mb:10}")
    private long maxSizeMb;

    @Value("${app.storage.allowed-types:image/jpeg,image/png,image/gif,application/pdf}")
    private String allowedTypesRaw;

    @Transactional
    public UploadedFile upload(MultipartFile file, Long memberId, String directory) {
        validate(file);

        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new IllegalArgumentException("Member not found: " + memberId));

        String storagePath = fileStorageService.store(file, directory);

        UploadedFile record = UploadedFile.builder()
            .member(member)
            .filename(extractFilename(storagePath))
            .originalFilename(file.getOriginalFilename())
            .contentType(file.getContentType())
            .sizeBytes(file.getSize())
            .storagePath(storagePath)
            .build();

        return uploadedFileRepository.save(record);
    }

    public void delete(Long fileId, Long requestingMemberId) {
        UploadedFile file = uploadedFileRepository.findById(fileId)
            .orElseThrow(() -> new IllegalArgumentException("File not found"));
        if (!file.getMember().getId().equals(requestingMemberId)) {
            throw new SecurityException("You do not own this file");
        }
        fileStorageService.delete(file.getStoragePath());
        uploadedFileRepository.delete(file);
    }

    public List<UploadedFile> listForMember(Long memberId) {
        return uploadedFileRepository.findByMemberId(memberId);
    }

    private void validate(MultipartFile file) {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("File is empty");
        }
        if (file.getSize() > maxSizeMb * 1024 * 1024) {
            throw new IllegalArgumentException("File exceeds maximum size of " + maxSizeMb + "MB");
        }
        Set<String> allowed = Set.of(allowedTypesRaw.split(","));
        if (!allowed.contains(file.getContentType())) {
            throw new IllegalArgumentException(
                "File type not allowed: " + file.getContentType() +
                ". Allowed: " + allowedTypesRaw
            );
        }
    }

    private String extractFilename(String storagePath) {
        int slash = storagePath.lastIndexOf('/');
        return slash >= 0 ? storagePath.substring(slash + 1) : storagePath;
    }
}
```

---

## Step 9 — `UploadedFileRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.UploadedFile;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface UploadedFileRepository extends JpaRepository<UploadedFile, Long> {
    List<UploadedFile> findByMemberId(Long memberId);
    List<UploadedFile> findByMemberIdOrderByCreatedAtDesc(Long memberId);
}
```

---

## Step 10 — `FileUploadController`

```java
package {{base_package}}.controller;

import {{base_package}}.service.FileUploadService;
import {{base_package}}.storage.FileStorageService;
import {{base_package}}.repository.UploadedFileRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.core.io.InputStreamResource;
import org.springframework.http.*;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.servlet.http.HttpSession;
import java.io.InputStream;

@Controller
@RequestMapping("/files")
@RequiredArgsConstructor
public class FileUploadController {

    private final FileUploadService fileUploadService;
    private final FileStorageService fileStorageService;
    private final UploadedFileRepository uploadedFileRepository;

    @GetMapping
    public String list(HttpSession session, Model model) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        model.addAttribute("files", fileUploadService.listForMember(memberId));
        return "member/files";
    }

    @PostMapping("/upload")
    public String upload(@RequestParam MultipartFile file,
                         @RequestParam(defaultValue = "general") String directory,
                         HttpSession session,
                         RedirectAttributes redirectAttributes) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        try {
            fileUploadService.upload(file, memberId, directory);
            redirectAttributes.addFlashAttribute("success", "File uploaded successfully.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/files";
    }

    @GetMapping("/download")
    public ResponseEntity<InputStreamResource> download(@RequestParam String path) {
        try {
            InputStream is = fileStorageService.retrieve(path);
            String filename = path.substring(path.lastIndexOf('/') + 1);
            return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
                .contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(new InputStreamResource(is));
        } catch (Exception e) {
            return ResponseEntity.notFound().build();
        }
    }

    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id,
                         HttpSession session,
                         RedirectAttributes redirectAttributes) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        try {
            fileUploadService.delete(id, memberId);
            redirectAttributes.addFlashAttribute("success", "File deleted.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/files";
    }
}
```

---

## Step 11 — `application.yml` additions

```yaml
app:
  storage:
    local:
      base-dir: ./uploads
    gcs:
      bucket-name: ${GCS_BUCKET_NAME:{{app_name_kebab}}-files}
    max-size-mb: 10
    allowed-types: "image/jpeg,image/png,image/gif,application/pdf,application/zip"

spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 15MB
```

---

## Step 12 — Add `uploads/` to `.gitignore`

```gitignore
uploads/
```

---

## Step 13 — Unit tests for validation logic

If `test_mode` is NOT `"token-save"`, write tests for `FileUploadService.validate()`:
- Empty file → `IllegalArgumentException`
- File exceeding max size → `IllegalArgumentException`
- Disallowed content type → `IllegalArgumentException`
- Valid file → passes

---

## Step 14 — Verify

Apply `test_mode`.

---

## Step 15 — Git commits

```bash
git add -A
git commit -m "feat: add file upload system with local and GCS storage backends"
```

---

## Step 16 — Update `installed_modules`

Add `"file-upload"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
File upload system is ready.

Local files are stored in ./uploads/
Profile active:
  local → LocalFileStorageService (./uploads/)
  prod  → GcsFileStorageService (GCS bucket)

Visit http://localhost:8080/files to upload a test file.
```

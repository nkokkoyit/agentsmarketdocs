# Agent Market Documentation

Tài liệu này mô tả dự án agent AI Recruitment System, tập trung vào kiến trúc, giao diện API, luồng xử lý và yêu cầu vận hành.

## Mục tiêu
- Giới thiệu tổng quan về hệ thống agent recruitment.
- Định nghĩa hợp đồng API và hành vi dịch vụ.
- Mô tả kiến trúc triển khai và luồng dữ liệu.
- Giải thích các bản ghi quyết định kỹ thuật (ADR), mô hình đe dọa và tuân thủ workflow.

## Tài liệu chính
- application-spec.md: đặc tả chức năng và phi chức năng của ứng dụng.
- api-spec.md: mô tả hợp đồng API, endpoint và mô hình dữ liệu.
- deployment-architecture.md: cấu trúc hệ thống, dịch vụ, và môi trường triển khai.
- sequence-flow.md: luồng tuần tự của tác vụ và xử lý bất đồng bộ.
- workflow-compliance.md: báo cáo tuân thủ quy trình và kết quả kiểm thử.
- threat-model.md: phân tích rủi ro và mô hình đe dọa.
- deployment-runbook.md: hướng dẫn vận hành và khởi chạy môi trường.
- PROJECT_ARCHITECTURE_BLUEPRINT.md: bản đồ kiến trúc tổng quan.
- redis-message-layout.md: định nghĩa mẫu tin Redis cho giao tiếp queue.
- adr-candidates.md: các ứng viên đề xuất cho ADR.
- release-notes.md: lịch sử phát hành và thay đổi chính.

## Sử dụng với MkDocs
- `mkdocs serve`: chạy site tài liệu cục bộ.
- `mkdocs build`: sinh site tĩnh vào thư mục `site/`.

---

### Ghi chú
- Cấu hình MkDocs đã được định nghĩa trong `mkdocs.yml`.
- Tài liệu này là điểm vào chính để điều hướng các bản mô tả dự án.

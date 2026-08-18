# Reflection — Day 18 Lakehouse Lab

**Học viên:** AICB-P2T2  
**Chủ đề:** Top 5 Lakehouse Anti-Patterns & Production Pitfalls

---

### Anti-pattern nhóm dễ vướng nhất: "Naive VACUUM & Blind Snapshot Expiry" (Storage Decay & Ghost Orphan Accumulation)

Trong 5 anti-pattern của kiến trúc Lakehouse (Small Files Syndrome, Missing Clustering/Z-Order, Schema Drift Chaos, Blind Expiry/Ghost Orphans, Dual-Storage Lifecycle Drift), nhóm chúng tôi dễ vướng nhất là **hiểu sai về cơ chế dọn dẹp (Maintenance Lifecycle) giữa transaction log và storage layer**.

#### Lý do:
1. **Lầm tưởng `VACUUM` dọn sạch mọi file rác:** Trong Delta Lake, lệnh `VACUUM` dựa vào transaction log để thu hồi các file đã bị tombstone sau khoảng retention. Tuy nhiên, các tiến trình streaming/ETL bị crash hoặc ngắt giữa chừng trước khi commit sẽ để lại các file data mồ côi (uncommitted orphans) chưa từng xuất hiện trong `_delta_log/`. Lệnh `VACUUM` mặc định hoàn toàn "mù" trước các file này, khiến dung lượng lưu trữ trên S3/Data Lake âm thầm phình to theo thời gian.
2. **`expire_snapshots` trong Iceberg chỉ cập nhật metadata:** Chạy expire snapshots không tự động xóa file Parquet vật lý nếu không kết hợp quét orphan data files (`remove_orphan_files`).

#### Giải pháp khắc phục:
Thiết lập định kỳ 4 job bảo trì bắt buộc: Compaction định kỳ, Clustering/Z-Order trên access patterns, Snapshot Expiry phối hợp cùng Sweep Orphan Files (quét sự khác biệt giữa storage và metadata/log), và Checkpointing log thường xuyên.

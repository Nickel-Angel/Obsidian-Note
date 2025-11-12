Vendor API Level 是指供应商分区支持的接口版本。该接口是双向的；它描述了 Vendor 分区向 System 分区提供的 API（HAL 接口），以及 Vendor 分区需要从 System 分区获得的 API（LLNDK 接口）。

Android 允许供应商将 Vendor 分区冻结在特定版本的 VSR 上。Android 期望 Vendor 分区提供的功能集由与 Vendor 分区关联的供应商 API 级别决定。
例如，如果 Vendor 分区的 API 级别不支持新功能，系统分区中的软件可能无法使用这些新功能。


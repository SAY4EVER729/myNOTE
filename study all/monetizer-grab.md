

```

graph TD
    A[grab入口] --> B[load_grab_configs]
    B --> C[grab_data]
    C --> D[grab_account_reports]
    C --> E[use_offline_data]
    C --> F1{是否当天数据?}
    F1 -->|是| G[grab_from_druid]
    F1 -->|否| H[grab_from_monetizer]
    H --> I[grab_reports_from_monetizer]
    I --> J[load_target_urls]
    I --> K[get具体平台grabber]
    K --> L[grabber.get]
```


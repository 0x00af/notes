# Thunderbird Date formatting

[support article](https://support.mozilla.org/en-US/kb/customize-date-time-formats-thunderbird) references the folloing table:

| Preference                                      | type   | default   | example    | value               | description |
| -------------------------------------------     | -------| --------- | ---------- | ------------------- | ----------- |
| intl.date_time.pattern_override.date_short      | string | (null)    | yyyy-MM-dd |          2025-12-31 | Short date  |
| intl.date_time.pattern_override.date_medium     | string | (null)    |            |                     | Medium date |
| intl.date_time.pattern_override.date_long       | string | (null)    |            |                     | Long date   |
| intl.date_time.pattern_override.date_full       | string | (null)    |            |                     | Full date   |
| intl.date_time.pattern_override.time_short      | string | (null)    |   HH:mm:ss |            09:59:31 | Short time  |
| intl.date_time.pattern_override.time_medium     | string | (null)    |            |                     | Medium time |
| intl.date_time.pattern_override.time_long       | string | (null)    |            |                     | Long time   |
| intl.date_time.pattern_override.time_full       | string | (null)    |            |                     | Full time   |
| intl.date_time.pattern_override.connector_short | string | (null)    |    {1} {0} | 2025-12-31 09:59:31 | formatting  |

This reflects in `prefs.js`:
```javascript
user_pref("intl.date_time.pattern_override.date_short", "yyyy-MM-dd");
user_pref("intl.date_time.pattern_override.time_short", "HH:mm:ss");
user_pref("intl.date_time.pattern_override.connector_short", "{1} {0}");
```

thunderbird doc references the unicode Date Field Symbol Table:

https://unicode.org/reports/tr35/tr35-dates.html#Date_Field_Symbol_Table

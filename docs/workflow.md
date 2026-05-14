# Detailed Pipeline Notes

## Data Sources & Schema

### booknow_booking.csv
- `book_theater_id` — theater identifier (BookNow system)
- `show_datetime` — date and time of the show
- `booking_datetime` — date and time the ticket was booked
- `tickets_booked` — number of tickets in that booking record

### booknow_theaters.csv
- `book_theater_id` — theater identifier
- `theater_type` — type of theater (Drama, etc.)
- `theater_area` — geographic area label
- `latitude`, `longitude` — geographic coordinates

### booknow_visits.csv
- `book_theater_id` — theater identifier
- `show_date` — date of the show
- `audience_count` — **TARGET VARIABLE** — actual daily audience

### cinePOS_booking.csv
- `cine_theater_id` — theater identifier (CinePOS POS system)
- `show_datetime` — show date and time
- `booking_datetime` — transaction date and time
- `tickets_sold` — tickets sold via POS

### cinePOS_theaters.csv
- `cine_theater_id` — theater identifier
- `theater_type`, `theater_area`, `latitude`, `longitude` — same schema

### movie_theater_id_relation.csv
- `book_theater_id` — BookNow theater ID
- `cine_theater_id` — corresponding CinePOS theater ID

### date_info.csv
- `show_date` — calendar date
- `day_of_week` — name of the weekday

---

## Merging Strategy

1. **BookNow pipeline:** `booknow_booking` ← inner join → `booknow_theaters` on `book_theater_id`
2. **CinePOS pipeline:** `cinePOS_booking` ← inner join → `cinePOS_theaters` on `cine_theater_id`
3. **Unify systems:** CinePOS merged gets `book_theater_id` via `movie_theater_id_relation`; `cine_theater_id` dropped; `tickets_sold` renamed to `tickets_booked`
4. **Concatenate** both booking streams → `all_bookings`
5. **Aggregate to daily:** Group by `[book_theater_id, show_date]`, sum `tickets_booked`, carry `theater_type`, `theater_area`, `latitude`, `longitude`
6. **Join ground truth:** `booknow_visits` ← left join → `daily_bookings` on `[book_theater_id, show_date]`
7. **Join calendar:** `final_df` ← left join → `date_info` on `show_date`

---

## Preprocessing Decisions

- **Missing `tickets_booked`:** Filled with 0 (no booking record = 0 tickets)
- **Missing `theater_type`, `theater_area`:** Filled with `"Unknown"` string
- **Missing `latitude`, `longitude`:** Filled with 0 (fallback; rare edge case)
- **LabelEncoder:** Fitted on `concat(train[col], val[col])` to prevent unseen-label errors at inference
- **Scaler:** `StandardScaler` applied only to numeric features; encoded categoricals passed through as integers

---

## Time-Based Split Rationale

Temporal ordering is preserved to prevent data leakage:
- Training set: first 80% of rows (sorted by `show_date`)
- Validation set: last 20% of rows (future dates)

This mirrors a real forecasting scenario — the model must predict future attendance from past patterns.

---

## Post-Processing Logic

After raw LightGBM predictions:

1. **Calibration** — Fit a `LinearRegression(val_pred → y_val)` and apply it to test predictions to correct systematic bias
2. **Theater-level bias correction** — Compute per-theater mean residual on validation, apply 40% of that bias to test predictions
3. **Baseline blend** — Final = `0.9 × bias_corrected + 0.1 × roll_7` (guards against theaters with no signal)
4. **Clip & round** — Clip to `[0, ∞)` and round to integer (audience count cannot be negative or fractional)

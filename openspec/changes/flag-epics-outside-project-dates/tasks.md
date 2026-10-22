## 1. Implementation
- [x] 1.1 `EpicListQuery` selects the project dates from its join (`project_start_date`, `project_end_date`)
- [x] 1.2 `EpicListTransformer` adds `outsideProjectDates` to each row
- [x] 1.3 `epics/list.blade.php` shows those dates in red with a screen-reader text and title
- [x] 1.4 "Outside the project dates" in es/eu/fr/en

## 2. Tests
- [x] 2.1 Feature test: an epic outside its project's dates is flagged; one inside and one without dates are not

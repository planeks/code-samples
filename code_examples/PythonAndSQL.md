# Python and SQL

## Dynamic SQL generation in Python

An example of the code that returns the list of columns of a table or material view. It is a part of the project that aggregates the data from the third-party databases where a lot of metrics fetches from the separate columns. The function is looking for the column names with specified template and generates the chart series labels based on the variable part of the name.

```python
def get_db_column_names(
        dataset_label: str, db_columns: list, db_name: str, db_table: str) -> list:
    """ Returns the list of tuples with column names and dataset labels for
    the given table.

    :param dataset_label: The label of dataset. Using asterisk (*) is supported
        for dynamic labels.
    :param db_columns: the list of DB columns. In most cases it can contains
        only single element.
    :param db_name: the name of the database connection.
    :param db_table: the name of the DB table.
    :return:
    """
    db_column_names = []
    db_column_labels = []

    for v in db_columns:
        if '%' in v:
            # We need to use different approaches for getting column names for tables and materialized views
            materialized_views = []
            with connections[db_name].cursor() as cursor:
                cursor.execute('SELECT matviewname FROM pg_matviews;')
                rows = cursor.fetchall()
            materialized_views = [x[0] for x in rows]

            if db_table in materialized_views:
                # Find column names with pattern for materialized view
                sql = f'SELECT a.attname FROM pg_attribute a JOIN pg_class t ON a.attrelid = t.oid '\
                      f'WHERE a.attnum > 0 AND NOT a.attisdropped '\
                      f'AND t.relname = \'{db_table}\' AND a.attname LIKE \'{v}\' ORDER BY a.attnum;'
            else:
                # Find column names with pattern for table
                sql = f'SELECT column_name FROM information_schema.columns WHERE '\
                      f'table_name = \'{db_table}\' AND column_name like \'{v}\''
            with connections[db_name].cursor() as cursor:
                cursor.execute(sql)
                rows = cursor.fetchall()

            column_name_chunks = v.split('%')
            for x in rows:
                db_column_names.append(x[0])
                d = x[0][len(column_name_chunks[0]):-len(column_name_chunks[1]) or None]
                if '*' in dataset_label:
                    db_column_labels.append(dataset_label.replace('*', d))
                else:
                    db_column_labels.append(dataset_label.format(value=d))
        else:
            db_column_names.append(v)
            db_column_labels.append(dataset_label)

    return db_column_names, db_column_labels
```

Fetches the timeseries data for the period of time with grouping data per day. The original database accumulates the metrics each hour as incremental values. The system collects the data for the end of the day and prepare it for the charts.

```python
    def get_dataset_for_vessel(
            self, noon_vessel: NoonVessel,
            start_date=None, finish_date=None, desc=False,
            limit=None, exclude_outliers: bool = False,
            report_type: Optional['ReportType'] = None,
    ):
        from datetime import date, timedelta

        finish_date = finish_date or date.today()
        start_date = start_date or (finish_date - timedelta(days=7))
        finish_date_str = finish_date.strftime('%Y-%m-%d')
        start_date_str = start_date.strftime('%Y-%m-%d')

        db_name = self.SOURCE_DB_NAME
        db_table = self.group.db_table
        db_column_names, db_column_labels = self.get_db_column_names()

        vessel_name = noon_vessel.name
        timestamp_column = 'de65_report_datetime_utc'
        vessel_column = 'de65_vessel_vesselname'
        report_type_column = 'de65_report_mode_desc'

        from core.utils import get_db_column_names
        x_db_columns = []
        if self.group.group_type == 'value_x_axis':
            x_db_columns, _ = get_db_column_names(
                dataset_label='x',
                db_columns=self.group.x_axis_db_columns.split(','),
                db_name=db_name,
                db_table=db_table,
            )
        threshold_db_columns = []
        if self.threshold_db_columns and self.threshold_condition:
            threshold_db_columns, _ = get_db_column_names(
                dataset_label='x',
                db_columns=self.threshold_db_columns.split(','),
                db_name=db_name,
                db_table=db_table,
            )

        order = 'ASC' if not desc else 'DESC'
        sql = 'SELECT '
        column_chunks = []
        for x in db_column_names:
            column_chunks.append(f'tt."{x}"')
            if self.incremental_value:
                column_chunks.append(f'tt."{x}" - LAG(tt."{x}") OVER (ORDER BY tt."{timestamp_column}") as "inc_{x}"')
        x_db_columns_shift = len(column_chunks)
        for x in x_db_columns:
            column_chunks.append(f'tt."{x}"')
        threshold_db_columns_shift = len(column_chunks)
        for x in threshold_db_columns:
            column_chunks.append(f'tt."{x}"')

        sql += ', '.join(column_chunks)

        sql += f', tt."{timestamp_column}"::date as datestamp'
        sql += f' FROM "{db_table}" tt'

        sql += f' INNER JOIN(SELECT "{timestamp_column}"::date, max("{timestamp_column}") as max_time FROM "{db_table}"'
        sql += f' WHERE "{timestamp_column}" >= \'{start_date_str}\' AND "{timestamp_column}" <= \'{finish_date_str}\''
        sql += f' AND "{vessel_column}" = \'{vessel_name}\''
        sql += f' GROUP BY "{timestamp_column}"::date ORDER BY "{timestamp_column}" {order}) groupedtt'
        sql += f' ON tt."{timestamp_column}" = groupedtt.max_time'

        sql += f' WHERE tt."{timestamp_column}" >= \'{start_date_str}\' AND tt."{timestamp_column}" <= \'{finish_date_str}\''
        sql += f' AND tt."{vessel_column}" = \'{vessel_name}\''
        if report_type is not None:
            sql += f' AND tt."{report_type_column}" = \'{report_type.name}\''
        if self.threshold_db_columns and self.threshold_condition:
            sql += ' AND ' + '+'.join([f'tt."{x}"' for x in threshold_db_columns]) + \
                   f' {self.threshold_condition} {self.threshold_value}'
        sql += f' ORDER BY tt."{timestamp_column}" {order}'
        if limit is not None:
            sql += f' LIMIT {limit} '
        sql += ';'

        with connections[db_name].cursor() as cursor:
            cursor.execute(sql)
            rows = cursor.fetchall()

        result = {
            'axe': self.unit.short_name if self.unit else 'y',
            'datasets': dict(),
            'columns': db_column_names,
            'table_col_count': 0,
        }

        for i, column in enumerate(db_column_names):
            _values = []
            for x in rows:
                if self.group.group_type == 'date_x_chart':
                    x_value = x[-1]
                else:
                    x_value = sum(x[x_db_columns_shift+i] or 0 for i, _ in enumerate(x_db_columns))
                y_value = x[i if not self.incremental_value else i*2+1]

                # Exclude outliers
                try:
                    if exclude_outliers and self.upper_limit_value is not None and y_value > self.upper_limit_value:
                        continue
                    if exclude_outliers and self.lower_limit_value is not None and y_value < self.lower_limit_value:
                        continue
                except TypeError:
                    # Catch the edge case with incorrect data in the DB
                    continue

                _values.append({
                    'x': x_value,
                    'y': y_value*self.y_value_coefficient if self.y_value_coefficient != 1.0 else y_value,
                })

            if self.group.group_type == 'value_x_axis':
                _values = sorted(_values, key=lambda k: k['x'])
            result['datasets'][column] = {
                'label': db_column_labels[i],
                'axe': self.unit.short_name if self.unit else 'y',
                'chart_type': self.chart_type,
                'values': _values,
                'position': self.position,
                'horizontal_position': self.horizontal_position,
                'hide_name': self.hide_name,
                'secondary_label': self.secondary_label,
                'extra_css': self.extra_css,
            }
            if self.horizontal_position > result['table_col_count']:
                result['table_col_count'] = self.horizontal_position

        return result
```



import pandas as pd
import psycopg2
import io

@data_exporter
def export_data(file_paths, *args, **kwargs):
    conn = psycopg2.connect(
        host="postgres", port=5432,
        dbname="pipeline_db", user="admin", password="admin5555"
    )
    cur = conn.cursor()
    cur.execute("TRUNCATE TABLE fact_readings")
    conn.commit()
    total = 0
    batch_size = 100

    for batch_start in range(0, len(file_paths), batch_size):
        batch = file_paths[batch_start:batch_start + batch_size]
        df_batch = pd.concat(
            [pd.read_parquet(fp) for fp in batch],
            ignore_index=True
        )
        buf = io.StringIO()
        df_batch[['department_name', 'sensor_serial', 'product_name',
                  'product_expire', 'create_at']].to_csv(buf, index=False, header=False)
        buf.seek(0)
        cur.copy_expert(
            "COPY fact_readings (department_name, sensor_serial, product_name, product_expire, create_at) FROM STDIN WITH CSV",
            buf
        )
        conn.commit()
        total += len(df_batch)
        del df_batch

        files_done = min(batch_start + batch_size, len(file_paths))
        if files_done % 5000 == 0 or files_done == len(file_paths):
            print(f"[{files_done}/{len(file_paths)}] {total:,} rows loaded")

    cur.close()
    conn.close()
    print(f"Finished: {total:,} rows loaded")
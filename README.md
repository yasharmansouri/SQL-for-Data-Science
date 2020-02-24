# SQL For Data Science:  

## PostgreSQL + Python PSYCOPG2

## Prepared by [Yashar Mansouri](www.https://www.linkedin.com/in/yasharmansouri)

The used database and the order of the material is based on Dan Sullivan's [Advanced SQL for Data Science](https://www.linkedin.com/learning/advanced-sql-for-data-scientists/window-functions-over-partition) LinkedIn Learning Course.

This is mostly the tuned up version with python implementation,  revised queries, and additional notes.

If you entered a wrong query and received an error, mak sure to close connection and open up again:  

- rememeber to change your arguements such as host, database name, password, port

- sql file with table creation and insertion is located in the **data** folder.

```python
cur.close()
conn.close()
conn = psycopg2.connect(host="localhost",database="data_sci", user="postgres", password="password", port=5432)
cur = conn.cursor()
```
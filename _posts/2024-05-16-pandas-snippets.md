---
layout: post
title: Pandas snippets
tags: python
---

Display rows where reply\_message=1
-----------------------------------

```text-plain
a.loc[a['reply_message'] == 1,['process_time', 'reply_message']]
```

Sort values
-----------

```text-plain
das = da.sort_values(by="timestamp")
dbs = db.sort_values(by="timestamp")
```

convert timestamp to epoch
--------------------------

```text-plain
da['timestamp'] = pd.to_datetime(da['event_time'] , errors='coerce').astype(int) // 10**9
```

rename 
-------

```text-plain
db.rename(columns={'calling_station': 'client_mac'}, inplace=True)
```

drop columns
------------

```text-plain
col_remove =  ['client_ip', 'user_id',
       'event_date', 'partition_date']
       
da = da.drop(auth_col_remove, axis=1) 
```

drop columns based on only what to keep
---------------------------------------

```text-plain
filter = auth[(auth['auth_name'] == 'test123')]
col_to_keep = ['category', 'ap_mac', 'auth_status', 'auth_client_mac', 'auth_event_time']
df_filtered = filter.drop(columns=filter.columns.difference(col_to_keep))
```

find column name where value appears
------------------------------------

This is useful when column names are different across data sets and you want to find out where the value occurs

```text-plain
files = [da,db,dc,dd]
final = {}

def find_df_name(df):
    name = [name for name, obj in globals().items() if id(obj) == id(df)]
    return name[0] if name else None
    
def find_row_with_value(df, value):
    name = find_df_name(df)
    for column in df.columns:
        if value in df[column].values:
            final[name] = column
            return df[df[column] == value]
            
# Call the function with your DataFrame and the value to search for

for i in files:
    result = find_row_with_value(i, 'testing1234')

print(final)
```

Display all rows where column value matches
-------------------------------------------

```text-plain
auth[(auth['user_id'] == '70a12345')]
```

multiple match conditions
-------------------------

```text-plain
df[(df['mac_addresss'] == 'aa:bb:02:03:04:05') & (df['op_status'] == 'ERROR')]
```

display unique elements in columns
----------------------------------

```text-plain
auth['mac_aes256'].unique()
```

display all columns 
--------------------

```text-plain
auth.columns
```

display shape
-------------

```text-plain
auth.shape
```

write to csv
------------

```text-plain
c.to_csv('output.csv', index=False)
```

where `c` is data frame

merge tables based on closest timestamp
---------------------------------------

```text-plain
# merge tables based on client_mac_aes256
mac = pd.merge_asof(nda, ndc, on="timestamp", by='client_mac', tolerance=300)
```

drop rows having columns NaN 
-----------------------------

```text-plain
mac_clean = mac.dropna(subset=['cal_timestamp_time', 'type_code', 'calling_address', 'called_address', 'ap_mac_address', 'ap_model'], how='all')
```

drop rows based on values of specific column
--------------------------------------------

```text-plain
tmp = auth.drop_duplicates(subset=['client_mac'])
```

find all values matching df1 in df2 
------------------------------------

```text-plain
nba_up_filtered = nnba_up[nnba_up["clientequipment_mac_address"].isin(list(unlval))]
```

Here check for `clientequipment_mac_address` from nnba\_up df  in unlval df 

type casting
------------

```text-plain
nnba_up = nba_up

nnba_up['upload_packets_count'] = nnba_up['upload_packets_count'].astype(int)
nnba_up['download_packets_count'] = nnba_up['download_packets_count'].astype(int)
```

convert string to int

calculate mean
--------------

```text-plain
mean_values = nba_up_filtered.groupby('clientequipment_mac_address_aes256').agg(
    {'upload_packets_count': 'mean',
     'download_packets_count': 'mean', 
     })
```

replace string to boolean
-------------------------

```text-plain
df_filtered['op_status'] = df_filtered['op_status'].replace({'SUCCESS': 1, 'FAILURE': 0, 'ERROR': 0})
```


# Разделение данных на заголовки и позиции

## Дано

Существует плоская таблица `lt_merged` содержащая поля как заголовка так и позиций, со структурой типа `ts_merge`:

```abap
    TYPES: BEGIN OF ts_group_key,
             carrid   TYPE scarr-carrid,
             carrname TYPE scarr-carrname,
           END OF ts_group_key.

    TYPES: BEGIN OF ts_totals,
             fltime   TYPE spfli-fltime,
             distance TYPE spfli-distance,
           END OF ts_totals.

    TYPES: BEGIN OF ts_header.
             INCLUDE TYPE ts_group_key AS group_key.
             INCLUDE TYPE ts_totals AS totals.
           TYPES:
                  END OF ts_header.

    TYPES: BEGIN OF ts_position,
             index    TYPE sy-index,
             carrid   TYPE spfli-carrid,
             connid   TYPE spfli-connid,
             fltime   TYPE spfli-fltime,
             distance TYPE spfli-distance,
           END OF ts_position.

    TYPES: BEGIN OF ts_merge,
             carrid   TYPE ts_header-carrid,
             carrname TYPE ts_header-carrname,
             connid   TYPE ts_position-connid,
             fltime   TYPE ts_position-fltime,
             distance TYPE ts_position-distance,
           END OF ts_merge.
```

## Задача

Используя возможности нового синтаксиса ABAP, cгруппировать данные в таблицу заголовков `lt_group` с типом строки `ts_group`, содержащую в поле `positions` вложенную таблицу позиций:

```abap
    TYPES tth_position TYPE HASHED TABLE OF ts_position WITH UNIQUE KEY carrid connid.

    TYPES: BEGIN OF ts_gheader.
             INCLUDE TYPE ts_header AS header.
           TYPES:
                    group_index TYPE int4,
                    group_size  TYPE int4,
                  END OF ts_gheader.

    TYPES: BEGIN OF ts_group.
             INCLUDE TYPE ts_gheader AS gheader.
           TYPES:
                    positions TYPE tth_position,
                  END OF ts_group.
```

## Решение

Используем оператор `VALUE output_type( FOR GROUPS ... )`:

```abap
    DATA(output_table) = VALUE output_type( FOR GROUPS group_name OF input_struct IN input_table 
        GROUP BY ( ... )  
        (
            ...
            positions_field = VALUE #( FOR input_struct2 IN GROUP group_name ( ... ) ) 
            ...
        ) 
    )
```

Для определения характеристик можно использовать ключевые слова `GROUP INDEX` и `GROUP SIZE`:

- `field1 = GROUP INDEX` - получение номера группы
- `field2 = GROUP SIZE` - получение размера группы

Полностью с подсуммированием полей `fltime` и `distance` для заголовка и заполнение номера позиции в поле `positions-index` (через `REDUCE #( ... )` ) эта конструкция может выглядеть так:

```abap
  DATA(lt_group) = VALUE tth_group(

    FOR GROUPS ls_group OF ls_merged IN lt_merged INDEX INTO l_ind

      GROUP BY (
        group_key   = VALUE ts_group_key(
          carrid   = ls_merged-carrid
          carrname = ls_merged-carrname
        )
        group_index = GROUP INDEX
        group_size  = GROUP SIZE
      )

      ( VALUE #( "VALUE #( ... ) is needed here to use LET ... IN operator
            group_index = ls_group-group_index
            group_size  = ls_group-group_size
            group_key   = ls_group-group_key
            totals      = REDUCE #(
              INIT ls_totals TYPE ts_totals
              FOR ls_grp IN GROUP ls_group
              NEXT ls_totals = VALUE #(
                fltime   = ls_totals-fltime + ls_grp-fltime
                distance = ls_totals-distance + ls_grp-distance
              )
            )
            positions   = REDUCE #(
              INIT lt_pos TYPE tt_position
              FOR ls_pos_merged IN GROUP ls_group
              NEXT lt_pos = VALUE #( BASE lt_pos
                (
                  index    = COND #( WHEN lt_pos IS INITIAL THEN 1 ELSE lt_pos[ lines( lt_pos ) ]-index + 1 )
                  carrid   = ls_pos_merged-carrid
                  connid   = ls_pos_merged-connid
                  fltime   = ls_pos_merged-fltime
                  distance = ls_pos_merged-distance
                )
              )
            )
      ) )
    ).
```

## Пример программы

Полный код примера см. программу [ztmp_sao_group_by_headers](https://github.com/arte0s/abap-examples/blob/main/src/ztmp_sao_group_by_headers.prog.abap)

### Входные данные (таблица lt_merged)

| CARRID | CARRNAME | CONNID | FLTIME | DISTANCE |
| --- | --- | --- | --- | --- |
| AA | American Airlines | 0017 | 361 | 2572.0 |
| AZ | Alitalia | 0555 | 125 | 845.0 |
| AZ | Alitalia | 0789 | 940 | 6130.0 |
| DL | Delta Airlines | 0106 | 475 | 3851.0 |
| JL | Japan Airlines | 0407 | 725 | 9100.0 |
| JL | Japan Airlines | 0408 | 675 | 9100.0 |
| LH | Lufthansa | 0400 | 444 | 6162.0 |
| LH | Lufthansa | 0401 | 435 | 6162.0 |
| LH | Lufthansa | 0402 | 455 | 6162.0 |
| LH | Lufthansa | 2402 | 65 | 555.0 |

### Результат группировки (таблица lt_group)

| CARRID | CARRNAME | FLTIME | DISTANCE | GROUP_INDEX | GROUP_SIZE | POSITIONS |
| --- | --- | --- | --- | --- | --- | --- |
| AA | American Airlines | 361 | 2572.0 | 1 | 1 | <table><thead><tr><th>INDEX</th><th>CARRID</th><th>CONNID</th><th>FLTIME</th><th>DISTANCE</th></tr></thead><tr><td>1</td><td>AA</td><td>0017</td><td>361</td><td>2572.0</td></tr></table> |
| AZ | Alitalia | 1065 | 6975.0 | 2 | 2 | <table><thead><tr><th>INDEX</th><th>CARRID</th><th>CONNID</th><th>FLTIME</th><th>DISTANCE</th></tr></thead><tr><td>1</td><td>AZ</td><td>0555</td><td>125</td><td>845.0</td></tr><tr><td>2</td><td>AZ</td><td>0789</td><td>940</td><td>6130.0</td></tr></table> |
| DL | Delta Airlines | 475 | 3851.0 | 3 | 1 | <table><thead><tr><th>INDEX</th><th>CARRID</th><th>CONNID</th><th>FLTIME</th><th>DISTANCE</th></tr></thead><tr><td>1</td><td>DL</td><td>0106</td><td>475</td><td>3851.0</td></tr></table> |
| JL | Japan Airlines | 1400 | 18200.0 | 4 | 2 | <table><thead><tr><th>INDEX</th><th>CARRID</th><th>CONNID</th><th>FLTIME</th><th>DISTANCE</th></tr></thead><tr><td>1</td><td>JL</td><td>0407</td><td>725</td><td>9100.0</td></tr><tr><td>2</td><td>JL</td><td>0408</td><td>675</td><td>9100.0</td></tr></table> |
| LH | Lufthansa | 1399 | 19041.0 | 5 | 4 | <table><thead><tr><th>INDEX</th><th>CARRID</th><th>CONNID</th><th>FLTIME</th><th>DISTANCE</th></tr></thead><tr><td>1</td><td>LH</td><td>0400</td><td>444</td><td>6162.0</td></tr><tr><td>2</td><td>LH</td><td>0401</td><td>435</td><td>6162.0</td></tr><tr><td>3</td><td>LH</td><td>0402</td><td>455</td><td>6162.0</td></tr><tr><td>4</td><td>LH</td><td>2402</td><td>65</td><td>555.0</td></tr></table> |

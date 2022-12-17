# Разделение данных на заголовки и позициии

## Дано

Существует плоская таблица `lt_merged` содержащая поля как заголовка так и позиций, со структурой типа `ts_merge`:

```abap
    TYPES: BEGIN OF ts_group_key,
             carrid   TYPE scarr-carrid,
             carrname TYPE scarr-carrname,
           END OF ts_group_key.

    TYPES: BEGIN OF ts_header.
             INCLUDE TYPE ts_group_key AS group_key.
           TYPES:
                    fltime   TYPE spfli-fltime,
                    distance TYPE spfli-distance,
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

Используя возможности нового синтаксиса ABAP, cгруппировать данные в таблицу заголовков `lt_group` с типом строки `ts_group`, содержашую в поле `positions` вложенную таблицу позиций:

```abap
    TYPES tth_position TYPE HASHED TABLE OF ts_position WITH UNIQUE KEY carrid connid.

    TYPES: BEGIN OF ts_grp,
             group_index TYPE int4,
             group_size  TYPE int4.
             INCLUDE TYPE ts_header AS header.
           TYPES:
                  END OF ts_grp.

    TYPES: BEGIN OF ts_group.
             INCLUDE TYPE ts_grp AS group.
           TYPES:
                    positions TYPE tth_position,
                  END OF ts_group.
```

## Решение

Используем оператор `VALUE output_type( FOR GROUPS ... ):

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

Полностью с подсуммированием полей `fltime` и `distance` для заголовока и заполнение номера позиции в поле `index` (через `REDUCE #( ... )` ) эта конструкция может выглядеть так:

```abap
    DATA(lt_group) = VALUE tts_group(

      FOR GROUPS ls_group OF ls_merged IN lt_merged INDEX INTO l_ind

        GROUP BY (
          group_index = GROUP INDEX
          group_size  = GROUP SIZE
          group_key   = VALUE ts_group_key(
            carrid   = ls_merged-carrid
            carrname = ls_merged-carrname
          )
        )

        ( VALUE #( "VALUE #( ... ) is needed here to use LET ... IN operator

            LET
              l_total_time = REDUCE ts_position-fltime(
                INIT l_time TYPE ts_position-fltime
                FOR ls_time_merged IN GROUP ls_group
                NEXT l_time = l_time + ls_time_merged-fltime
              )

              l_total_distance = REDUCE ts_position-fltime(
                INIT l_distance TYPE ts_position-fltime
                FOR ls_dist_merged IN GROUP ls_group
                NEXT l_distance = l_distance + ls_dist_merged-distance
              )

            IN
              group_index = ls_group-group_index
              group_size  = ls_group-group_size
              group_key   = ls_group-group_key
              fltime      = l_total_time
              distance    = l_total_distance
***              positions   = VALUE #( FOR ls_pos_merged IN GROUP ls_group INDEX INTO l_pos_index "INDEX INTO doesn't work!
              positions   = REDUCE #(
                INIT lt_pos TYPE tts_position
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

Полный код примера см. в [abap-examples/src/ztmp_sao_group_by_headers.prog.abap](https://github.com/arte0s/abap-examples/blob/main/src/ztmp_sao_group_by_headers.prog.abap)

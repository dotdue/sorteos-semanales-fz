## Discord: `@dois.`
### Compilado en GM EvertZone, MySQL plugin R39-4
### Necesario nivel básico de Pawn
![screenshot](https://github.com/dotdue/sorteos-semanales-fz/blob/main/screenshot.png?raw=true)

# SQL
Crear una columna en la tabla `usuarios` para guardar el tiempo jugado.
```sql
ALTER TABLE usuarios ADD COLUMN weekly_playtime_seconds INT DEFAULT 0;
```
Crear una tabla para guardar los jugadores que ganaron el sorteo.
```sql
CREATE TABLE `users_weekly_draw` (
	`user_id` INT(10) UNSIGNED NULL DEFAULT NULL,
	`prize_received` TINYINT(3) NULL DEFAULT '0'
);
```
Crear una tabla con la fecha de cuándo ocurrirá el sorteo.
```sql
CREATE TABLE `server_settings` (
	`draw_date` DATE NULL DEFAULT NULL
);
```
La columna `draw_date` será la fecha del sorteo, debes elegir una fecha y en esa fecha ocurrirá el sorteo.  
Cuando ocurra, se sumarán 7 días (una semana) para que el sorteo ocurra nuevamente.  
Ejemplo: 14/12/2024 (Sábado de diciembre)  
El sorteo ocurrirá en esa fecha y luego el 21/12/2024 (Sábado de diciembre, 7 días después de la fecha anterior).
```sql
INSERT INTO `server_settings` (`draw_date`) VALUES ('2024-12-14');
```

# Pawn
#### Variables (HEADER)
```c
enum
{
    QUERY_DRAWING_GET_DATE,
    QUERY_DRAWING_GET_WINNERS,
    QUERY_DRAWING_WINNER_GIVE_PRIZE,
}

enum E_SERVER_DRAWING
{
    bool:DRAW_UPDATING,
    DRAW_DATE[10 + 1],
}
new
    SERVER_DRAWING[E_SERVER_DRAWING],
    g_PlayerPlayingTime[MAX_PLAYERS];
```

#### Para las publics que ya existen.
```c
public OnGameModeInit()
{
    Drawing_Init();
    return 1;
}

public OnPlayerSpawn(playerid)
{
    if (PrimerSpawn[playerid] == 0)
    {
        static query[148];
        mysql_format(db_handle, query, sizeof query, "SELECT `user_id` FROM `users_weekly_draw` WHERE EXISTS (SELECT 1 FROM `users_weekly_draw` WHERE `user_id` = %i AND `prize_received` = 0);", PlayerInfo[playerid][pID]);
        mysql_tquery(db_handle, query, "OnQueryDrawing", "ii", QUERY_DRAWING_WINNER_GIVE_PRIZE, playerid);

        g_PlayerPlayingTime[playerid] = gettime();
    }
    return 1;
}

public OnPlayerDisconnect(playerid)
{
    if (JugadorLogeado[playerid] == 1)
    {
        static query[118];
        mysql_format(db_handle, query, sizeof query, "UPDATE `usuarios` SET `weekly_playtime_seconds` = `weekly_playtime_seconds` + %i WHERE `ID` = %i;", gettime() - g_PlayerPlayingTime[playerid], PlayerInfo[playerid][pID]);
        mysql_tquery(db_handle, query, "OnQueryDrawing");

        g_PlayerPlayingTime[playerid] = 0;
    }
    return 1;
}
```

#### Sistema
```c
Drawing_Init()
{
    Drawing_Refresh();
    SetTimer("TimerOneSecondInterval", 1000, true);
    return 1;
}

Drawing_Refresh(bool:update_last_draw = false)
{
    if (SERVER_DRAWING[DRAW_UPDATING])
        return 0;

    SERVER_DRAWING[DRAW_UPDATING] = true;

    if (update_last_draw)
        mysql_tquery(db_handle, "UPDATE `server_settings` SET `draw_date` = DATE_ADD(CURDATE(), INTERVAL 7 DAY);");
    
    mysql_tquery(db_handle, "SELECT `draw_date` FROM `server_settings`;", "OnQueryDrawing", "i", QUERY_DRAWING_GET_DATE);
    return 1;
}

Drawing_Winners()
{
    /*
        3600 = 1 hora
    */
    mysql_tquery(db_handle, "TRUNCATE TABLE `users_weekly_draw`;");
    mysql_tquery(db_handle, "\
        INSERT INTO `users_weekly_draw` (user_id) \
            SELECT `ID` \
        FROM \
            `usuarios` \
        WHERE \
            `weekly_playtime_seconds` >= 3600 \
        ORDER BY RAND() \
            LIMIT 10;"
    );

    mysql_tquery(db_handle, "UPDATE `usuarios` SET `weekly_playtime_seconds` = 0;");
    mysql_tquery(db_handle, "SELECT `user_id` FROM `users_weekly_draw`;", "OnQueryDrawing", "i", QUERY_DRAWING_GET_WINNERS);
    return 1;
}

Drawing_GivePrize(playerid, prize)
{
    if (!IsPlayerConnected(playerid))
        return 0;

    if (!JugadorLogeado[playerid])
        return 0;

    static query[80 + 1];
    mysql_format(db_handle, query, sizeof query, "UPDATE `users_weekly_draw` SET `prize_received` = 1 AND `user_id` = %i;", PlayerInfo[playerid][pID]);
    mysql_pquery(db_handle, query);

    switch (prize)
    {
        case 1: PlayerInfo[playerid][Totems] += 3;
        case 2: PlayerInfo[playerid][Totems] += 2;
        case 3: PlayerInfo[playerid][Moneda] += 15;
        case 4: PlayerInfo[playerid][Moneda] += 10;
        case 5: PlayerInfo[playerid][Totems] += 1;
        case 6: SetPlayerVIP(playerid, .type = 2, .days = 30);
        case 7: PlayerInfo[playerid][Moneda] += 5;
        case 8: SetPlayerVIP(playerid, .type = 2, .days = 15);
        case 9: SetPlayerVIP(playerid, .type = 2, .days = 7);
        case 10: PlayerInfo[playerid][jDinero] += 500000;
    }
    return 1;
}

GetPlayerFromDBID(user_id)
{
    foreach (new playerid : Player)
    {
        if (PlayerInfo[playerid][pID] == user_id)
            return playerid;
    }

    return INVALID_PLAYER_ID;
}

forward OnQueryDrawing(params, extraid);
public OnQueryDrawing(params, extraid)
{
    switch (params)
    {
        case QUERY_DRAWING_GET_DATE:
        {
            if (cache_num_rows())
            {
                cache_get_field_content(0, "draw_date", SERVER_DRAWING[DRAW_DATE]);
                SERVER_DRAWING[DRAW_UPDATING] = false;
            }
        }
        case QUERY_DRAWING_GET_WINNERS:
        {
            new
                const rows = cache_num_rows();

            for (new row; row < rows; row++)
            {
                extraid = GetPlayerFromDBID(cache_get_row_int(row, 0));
                Drawing_GivePrize(extraid, row + 1);
            }
        }
        case QUERY_DRAWING_WINNER_GIVE_PRIZE:
        {
            new
                const rows = cache_num_rows();

            for (new row; row < rows; row++)
            {
                if (!IsPlayerConnected(extraid))
                    break;

                if (cache_get_row_int(row, 0) == PlayerInfo[extraid][pID])
                {
                    Drawing_GivePrize(extraid, row + 1);
                    break;
                }
            }
        }
    }
    return 1;
}

forward TimerOneSecondInterval();
public TimerOneSecondInterval()
{
    static
        current_date[10 + 1], year, month, day;

    getdate(year, month, day);
    format(current_date, sizeof current_date, "%02d-%02d-%02d", year, month, day);

    if (!SERVER_DRAWING[DRAW_UPDATING])
    {
        if (!strcmp(current_date, SERVER_DRAWING[DRAW_DATE]))
        {
            Drawing_Refresh(true);
            Drawing_Winners();
        }
    }
    return 1;
}
```

#### Dialog ganadores
```c
forward ShowDrawingDialog(playerid);
public ShowDrawingDialog(playerid)
{
    if (!IsPlayerConnected(playerid))
        return 0;

    static dialog[580 + 1], username[MAX_PLAYER_NAME + 1], fmat[58 + 1];
    dialog[0] = '\0';

    new
        const rows = cache_num_rows();

    if (!rows)
        return ShowPlayerDialog(playerid, 32760, DIALOG_STYLE_MSGBOX, "Ganadores del último sorteo", "{FFFFFF}Sin ganadores.", "Ok", #);

    for (new row = 0, bool:received; row < rows; row++)
    {
        cache_get_field_content(row, "username", username);
        received = bool:(cache_get_field_content_int(row, "prize_received"));

        switch ((row + 1))
        {
            case 1: fmat = "3 Totems";
            case 2: fmat = "2 Totems";
            case 3: fmat = "15 Fz";
            case 4: fmat = "10 Fz";
            case 5: fmat = "1 Totem";
            case 6: fmat = "30 días de VIP2";
            case 7: fmat = "5 Fz";
            case 8: fmat = "15 días de VIP2";
            case 9: fmat = "7 días de VIP2";
            case 10: fmat = "$500.000";
        }
        format(fmat, sizeof fmat, "%s\t%s (%s)\n", username, fmat, received ? "Recebido" : "Pendente");
        strcat(dialog, fmat);
    }
    ShowPlayerDialog(playerid, 32760, DIALOG_STYLE_TABLIST, "Ganadores del último sorteo", dialog, "Ok", #);
    return 1;
}

CMD:sorteo(playerid)
{
    mysql_tquery(db_handle, "\
        SELECT \
            u.username, \
            draw.prize_received \
        FROM \
            users_weekly_draw draw \
        JOIN \
            usuarios u \
        ON \
            draw.user_id = u.ID;",
        "ShowDrawingDialog", "i", playerid
    );
    return 1;
}
```

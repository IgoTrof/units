-------------- CalcListService ------------
package mashstroi.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import mashstroi.dto.CalcListDto;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class CalcListService {

    private final JdbcTemplate jdbcTemplate;

    // Базовый SQL из твоего Delphi-кода
    private static final String BASE_SQL = 
            "select c.CALC_ID, c.NAME, cast(c.DATE_INPUT as date) as DATE_INPUT, " +
            "mu.USER_FULL_NAME, c.NUM_DRAFT, c.NUM_TP, te.NAME as NAMEENTERP, " +
            "(case when c.SOST_ID = 10 then 'создан' " +
            "      when c.SOST_ID = 20 then 'проверка' " +
            "      when c.SOST_ID = 25 then 'согласовано' " +
            "      when c.SOST_ID = 30 then 'утвержден' " +
            "      when c.SOST_ID = 50 then 'майбюн' end) as SOST_NAME " +
            "from mech.CALC c " +
            "left join mech.TNAR_ENTERP te on te.ENTERP_ID = c.ENTERP_ID " +
            "left join mech.MECH_USER mu on mu.USER_ID = c.USER_ID " +
            "where 1=1 ";

    // Маппер: превращает строку из БД в наш DTO
    private final RowMapper<CalcListDto> rowMapper = new RowMapper<>() {
        @Override
        public CalcListDto mapRow(ResultSet rs, int rowNum) throws SQLException {
            CalcListDto dto = new CalcListDto();
            dto.setCalcId(rs.getBigDecimal("CALC_ID"));
            dto.setName(rs.getString("NAME"));
            
            // Безопасно достаем дату (может быть null)
            java.sql.Date sqlDate = rs.getDate("DATE_INPUT");
            dto.setDateInput(sqlDate != null ? sqlDate.toLocalDate() : null);
            
            dto.setUserFullName(rs.getString("USER_FULL_NAME"));
            dto.setNumDraft(rs.getString("NUM_DRAFT"));
            dto.setNumTp(rs.getString("NUM_TP"));
            dto.setNameEnterp(rs.getString("NAMEENTERP"));
            dto.setSostName(rs.getString("SOST_NAME"));
            return dto;
        }
    };

    @Transactional // Важно для работы SessionContextAspect!
    public List<CalcListDto> getCalcList(Map<String, String> filters) {
        // Используем StringBuilder для динамического WHERE
        StringBuilder sql = new StringBuilder(BASE_SQL);
        List<Object> params = new ArrayList<>();

        // 1. Фильтрация по параметрам от пользователя (если фронт прислал)
        if (filters.containsKey("name") && filters.get("name") != null && !filters.get("name").isBlank()) {
            sql.append(" AND c.NAME ILIKE ? ");
            params.add("%" + filters.get("name") + "%");
        }
        
        if (filters.containsKey("nameEnterp") && filters.get("nameEnterp") != null && !filters.get("nameEnterp").isBlank()) {
            sql.append(" AND te.NAME ILIKE ? ");
            params.add("%" + filters.get("nameEnterp") + "%");
        }

        // 2. Сортировка (по умолчанию по ID)
        sql.append(" ORDER BY c.CALC_ID DESC ");

        log.info("Выполнение запроса списка расчетов. SQL: {}", sql.toString());
        
        // Выполняем запрос с параметрами (защита от SQL-инъекций)
        return jdbcTemplate.query(sql.toString(), rowMapper, params.toArray());
    }
}

---------------------------- CalcController ------------------------------

package mashstroi.controller;

import lombok.RequiredArgsConstructor;
import mashstroi.dto.CalcListDto;
import mashstroi.service.CalcListService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/calc-list")
@RequiredArgsConstructor
public class CalcListController {

    private final CalcListService calcListService;

    // Вызов: GET /api/calc-list?username=USERNAME_FIO&name=Крышка
    @GetMapping
    public List<CalcListDto> getList(@RequestParam Map<String, String> filters) {
        return calcListService.getCalcList(filters);
    }
}


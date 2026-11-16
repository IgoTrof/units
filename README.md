
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

    // Вызов: GET /api/calc-list?username=your_login&name=Крышка
    @GetMapping
    public List<CalcListDto> getList(@RequestParam Map<String, String> filters) {
        return calcListService.getCalcList(filters);
    }
}


2 CRUD
---------------------------- CalcCrudDto.java в пакете dto -----------
package mashstroi.dto;

import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDate;

@Data
public class CalcCrudDto {
    private BigDecimal calcId;       // PK. При создании может быть null, база/последовательность выдаст сама
    private String name;             // Наименование изделия
    private LocalDate dateInput;     // Дата составления
    private String numDraft;         // Номер чертежа
    private String numTp;            // Номер техпроцесса
    private BigDecimal enterpId;     // ID подразделения
    private BigDecimal vidZagot;     // Справочник: вид заготовки
    private BigDecimal materU;       // Справочник: материал
    private BigDecimal userCount;    // Тот самый user_id или user_count из базы
    
    // ... добавь остальные поля из своей таблицы, если нужно
}
-------------------------- CalcCrudService.java -----------------
package mashstroi.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import mashstroi.dto.CalcCrudDto;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.namedparam.BeanPropertySqlParameterSource;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class CalcCrudService {

    private final NamedParameterJdbcTemplate namedJdbcTemplate;

    // 1. ПОЛУЧИТЬ ОДНУ ЗАПИСЬ ПО ID (Read)
    @Transactional
    public CalcCrudDto getById(BigDecimal calcId) {
        String sql = "SELECT * FROM mech.calc WHERE calc_id = :id";
        
        // MapSqlParameterSource - это просто способ передать параметры (как Map)
        MapSqlParameterSource params = new MapSqlParameterSource("id", calcId);
        
        List<CalcCrudDto> list = namedJdbcTemplate.query(sql, params, 
                new BeanPropertyRowMapper<>(CalcCrudDto.class));
                
        return list.isEmpty() ? null : list.get(0);
    }

    // 2. СОЗДАТЬ (Create)
    @Transactional
    public void create(CalcCrudDto dto) {
        // ВАЖНО: В легаси-базах ID часто генерируется через sequence. 
        // Если у тебя sequence называется иначе, поменяй 'mech.calc_seq'
        // Если ID приходит с фронта, просто используй :calcId в INSERT
        String sql = "INSERT INTO mech.calc " +
                "(calc_id, name, date_input, num_draft, num_tp, enterp_id, vid_zagot, mater_u) " +
                "VALUES (nextval('mech.calc_seq'), :name, :dateInput, :numDraft, :numTp, :enterpId, :vidZagot, :materU)";

        // BeanPropertySqlParameterSource берет все поля из DTO и подставляет их в :name, :dateInput и т.д.
        namedJdbcTemplate.update(sql, new BeanPropertySqlParameterSource(dto));
        log.info("Создан новый расчет: {}", dto.getName());
    }

    // 3. ОБНОВИТЬ (Update)
    @Transactional
    public void update(CalcCrudDto dto) {
        String sql = "UPDATE mech.calc SET " +
                "name = :name, " +
                "date_input = :dateInput, " +
                "num_draft = :numDraft, " +
                "num_tp = :numTp, " +
                "enterp_id = :enterpId, " +
                "vid_zagot = :vidZagot, " +
                "mater_u = :materU " +
                "WHERE calc_id = :calcId";

        namedJdbcTemplate.update(sql, new BeanPropertySqlParameterSource(dto));
        log.info("Обновлен расчет с ID: {}", dto.getCalcId());
    }

    // 4. УДАЛИТЬ (Delete)
    @Transactional
    public void delete(BigDecimal calcId) {
        String sql = "DELETE FROM mech.calc WHERE calc_id = :id";
        MapSqlParameterSource params = new MapSqlParameterSource("id", calcId);
        namedJdbcTemplate.update(sql, params);
        log.info("Удален расчет с ID: {}", calcId);
    }
}
--------------- CalcCrudController.java --
package mashstroi.controller;

import lombok.RequiredArgsConstructor;
import mashstroi.dto.CalcCrudDto;
import mashstroi.service.CalcCrudService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;

@RestController
@RequestMapping("/api/calc-crud")
@RequiredArgsConstructor
public class CalcCrudController {

    private final CalcCrudService calcCrudService;

    // GET: Получить расчет для редактирования
    @GetMapping("/{id}")
    public ResponseEntity<CalcCrudDto> getById(@PathVariable BigDecimal id) {
        CalcCrudDto dto = calcCrudService.getById(id);
        if (dto == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(dto);
    }

    // POST: Создать новый расчет
    @PostMapping
    public ResponseEntity<Void> create(@RequestBody CalcCrudDto dto) {
        calcCrudService.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    // PUT: Обновить расчет
    @PutMapping("/{id}")
    public ResponseEntity<Void> update(@PathVariable BigDecimal id, @RequestBody CalcCrudDto dto) {
        dto.setCalcId(id); // Гарантируем, что обновляем тот ID, который в URL
        calcCrudService.update(dto);
        return ResponseEntity.ok().build();
    }

    // DELETE: Удалить расчет
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable BigDecimal id) {
        calcCrudService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
------------ bush ------
curl -X PUT "http://localhost:8090/api/calc-crud/1?username=твой_логин" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "Крышка через Bash",
           "dateInput": "2026-11-15",
           "numDraft": "999",
           "numTp": "111"
         }'
         
------------ console -----
javascript
fetch('http://localhost:8090/api/calc-crud/1?username=твой_логин', {
    method: 'PUT',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        "name": "Крышка (обновленная)",
        "dateInput": "2026-11-15",
        "numDraft": "123-456",
        "numTp": "777",
        "enterpId": 1,
        "vidZagot": 2,
        "materU": 3
    })
})
.then(response => {
    if (response.ok) console.log("✅ УСПЕХ! Данные обновлены.");
    else console.log("❌ ОШИБКА! Статус: " + response.status);
})
.catch(error => console.error("❌ Ошибка сети:", error));
------------ 

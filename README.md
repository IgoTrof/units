
        // 3. Динамически добавляем условия (Используем ILIKE для нечувствительности к регистру)
        if (filters.containsKey("name") && filters.get("name") != null && !filters.get("name").isBlank()) {
            sqlBuilder.append(" AND c.name ILIKE :name");
            params.put("name", "%" + filters.get("name") + "%"); // % для поиска по частичному совпадению
        }
        if (filters.containsKey("num_draft") && filters.get("num_draft") != null && !filters.get("num_draft").isBlank()) {
            sqlBuilder.append(" AND c.num_draft ILIKE :num_draft");
            params.put("num_draft", "%" + filters.get("num_draft") + "%");
        }
        if (filters.containsKey("nm_rol") && filters.get("nm_rol") != null && !filters.get("nm_rol").isBlank()) {
            sqlBuilder.append(" AND r.name ILIKE :nm_rol");
            params.put("nm_rol", "%" + filters.get("nm_rol") + "%");
        }
        if (filters.containsKey("user_full_name") && filters.get("user_full_name") != null && !filters.get("user_full_name").isBlank()) {
            sqlBuilder.append(" AND u.user_full_name ILIKE :user_full_name");
            params.put("user_full_name", "%" + filters.get("user_full_name") + "%");
        }
        
        // ФИЛЬТР ПО СТАТУСАМ (Сложный момент, так как в SELECT это текст, а в БД ID)
        // Фронт пришлет "создан", а нам надо отфильтровать по l.NEW_SOST_ID = 10
        if (filters.containsKey("new_sost") && filters.get("new_sost") != null && !filters.get("new_sost").isBlank()) {
            sqlBuilder.append(" AND l.NEW_SOST_ID = :new_sost_id");
            params.put("new_sost_id", getSostIdByName(filters.get("new_sost"))); 
        }
        if (filters.containsKey("old_sost") && filters.get("old_sost") != null && !filters.get("old_sost").isBlank()) {
            sqlBuilder.append(" AND l.OLD_SOST_ID = :old_sost_id");
            params.put("old_sost_id", getSostIdByName(filters.get("old_sost")));
        }

        // 4. Создаем запрос
        @SuppressWarnings("unchecked")
        var query = entityManager.createNativeQuery(sqlBuilder.toString());
        
        // 5. Подставляем параметры
        for (Map.Entry<String, Object> entry : params.entrySet()) {
            query.setParameter(entry.getKey(), entry.getValue());
        }

        // 6. Выполняем и маппим (как мы делали раньше)
        List<Object[]> rows = query.getResultList();
        List<CalcSostListDto> result = new ArrayList<>();
        
        for (Object[] row : rows) {
            CalcSostListDto dto = new CalcSostListDto(
                    row[0] != null ? ((Number) row[0]).longValue() : null,
                    row[1] != null ? ((Number) row[1]).longValue() : null,
                    (String) row[2],
                    (String) row[3],
                    (String) row[4],
                    (String) row[5],
                    (String) row[6],
                    row[7] != null ? row[7].toString() : null,
                    (String) row[8],
                    (String) row[9]
            );
            result.add(dto);
        }
        return result;
    }

    // Вспомогательный метод для перевода текста статуса в его ID из БД
    private Integer getSostIdByName(String statusName) {
        return switch (statusName.toLowerCase()) {
            case "создан" -> 10;
            case "проверка" -> 20;
            case "согласовано" -> 25;
            case "утвержден" -> 30;
            default -> -1; // Если передали что-то неизвестное
        };
    }

-- 1. Перша сесія акаунта (без дублювання)
WITH account_first_session AS (
    SELECT
        account_id,
        registration_date,
        country
    FROM (
        SELECT
            acs.account_id,
            CAST(s.date AS DATE) AS registration_date,
            sp.country,
            ROW_NUMBER() OVER (PARTITION BY acs.account_id ORDER BY s.date) AS rn
        FROM `data-analytics-mate.DA.account_session` acs
        JOIN `data-analytics-mate.DA.session` s
            ON acs.ga_session_id = s.ga_session_id
        JOIN `data-analytics-mate.DA.session_params` sp
            ON s.ga_session_id = sp.ga_session_id
    )
    WHERE rn = 1
),


-- 2. Метрики по акаунтам
base_accounts AS (
    SELECT
        f.registration_date AS date,
        f.country,
        a.send_interval,
        CASE WHEN a.is_verified = 1 THEN 'verified' ELSE 'not_verified' END AS is_verified,
        CASE WHEN a.is_unsubscribed = 1 THEN 'unsubscribed' ELSE 'subscribed' END AS is_unsubscribed,
        COUNT(DISTINCT a.id) AS account_cnt,
        0 AS sent_msg,
        0 AS open_msg,
        0 AS visit_msg
    FROM `data-analytics-mate.DA.account` a
    JOIN account_first_session f
        ON a.id = f.account_id
    GROUP BY 1,2,3,4,5
),


-- 3. Метрики по email
base_emails AS (
    SELECT
        DATE_ADD(f.registration_date, INTERVAL es.sent_date DAY) AS date,
        f.country,
        a.send_interval,
        CASE WHEN a.is_verified = 1 THEN 'verified' ELSE 'not_verified' END AS is_verified,
        CASE WHEN a.is_unsubscribed = 1 THEN 'unsubscribed' ELSE 'subscribed' END AS is_unsubscribed,
        0 AS account_cnt,
        COUNT(DISTINCT es.id_message) AS sent_msg,
        COUNT(DISTINCT eo.id_message) AS open_msg,
        COUNT(DISTINCT ev.id_message) AS visit_msg
    FROM `data-analytics-mate.DA.email_sent` es
    JOIN `data-analytics-mate.DA.account` a
        ON es.id_account = a.id
    JOIN account_first_session f
        ON a.id = f.account_id
    LEFT JOIN `data-analytics-mate.DA.email_open` eo
        ON es.id_message = eo.id_message
    LEFT JOIN `data-analytics-mate.DA.email_visit` ev
        ON es.id_message = ev.id_message
    GROUP BY 1,2,3,4,5
),


-- 4. UNION + агрегація
unioned AS (
    SELECT * FROM base_accounts
    UNION ALL
    SELECT * FROM base_emails
),


aggregated AS (
    SELECT
        date,
        country,
        send_interval,
        is_verified,
        is_unsubscribed,
        SUM(account_cnt) AS account_cnt,
        SUM(sent_msg) AS sent_msg,
        SUM(open_msg) AS open_msg,
        SUM(visit_msg) AS visit_msg
    FROM unioned
    GROUP BY 1,2,3,4,5
),


-- 5. Тотали по країні (віконка)
final_with_totals AS (
    SELECT
        *,
        SUM(account_cnt) OVER (PARTITION BY country) AS total_country_account_cnt,
        SUM(sent_msg) OVER (PARTITION BY country) AS total_country_sent_cnt
    FROM aggregated
),


-- 6. Ранги
final_ranked AS (
    SELECT
        *,
        DENSE_RANK() OVER (ORDER BY total_country_account_cnt DESC) AS rank_total_country_account_cnt,
        DENSE_RANK() OVER (ORDER BY total_country_sent_cnt DESC) AS rank_total_country_sent_cnt
    FROM final_with_totals
)


-- 7. Фінал
SELECT *
FROM final_ranked
WHERE rank_total_country_account_cnt <= 10
   OR rank_total_country_sent_cnt <= 10
ORDER BY date DESC, country;

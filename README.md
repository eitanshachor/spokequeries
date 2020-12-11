# Spoke Queries
An updating list for keeping Spoke queries.

- [Campaign Contacts Who Opted-Out w/'STOP'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#campaign-contacts-who-opted-out-wstop)
- [Distinct Cells Where People Opted Out w/'Stop'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#distinct-cells-where-people-opted-out-wstop)
- [Proper Date Formatting](https://github.com/eitanshachor/spokequeries/blob/main/README.md#proper-date-formatting)
- [New Voters Contacted Each Day](https://github.com/eitanshachor/spokequeries/blob/main/README.md#new-voters-contacted-each-day)
- [Sent Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#sent-texts)
- [Received Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#received-texts)
***

## Topline Numbers w/Query Links

| Metric | Number | Updated |
| :--- | ---: | ---: |
| [Sent Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#sent-texts) | `3,203,204` | 12/10/20 |
| [Received Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#received-texts) | `515,111` | 12/10/20 |


***
### Campaign Contacts Who Opted-Out w/'STOP'
```SQL
SELECT 
	COUNT(cc.id)
FROM 
	campaign_contact as cc
JOIN
	campaign ON campaign.id = cc.campaign_id
WHERE
	error_code = -133
	AND campaign.is_started = 'TRUE'
	AND campaign.organization_id = 5
	AND campaign.id > 21
```
Count on 12/10/20: `235,349`

### Distinct Cells Where People Opted Out w/'Stop'
```SQL
SELECT 
	COUNT(DISTINCT(cc.cell))
FROM 
	campaign_contact as cc
JOIN
	campaign ON campaign.id = cc.campaign_id
WHERE
	error_code = -133
	AND campaign.is_started = 'TRUE'
	AND campaign.organization_id = 5
	AND campaign.id > 21
```
Count on 12/10/20: `217,695`

### Proper Date Formatting
```SQL
to_char(u.created_at AT time zone 'mst', 'FMDy, FMMon FMDD, YYYY at FMHH:MI PM') as join_date
```
### New Voters Contacted Each Day
```SQL
SELECT 
	COUNT(cell_number) as new_voters_contacted, 
	contact_date
FROM (
	SELECT 
		message.contact_number as cell_number, 
		to_char(MIN(message.created_at), 'MM-DD-YY') as contact_date
	FROM 
		message
	INNER JOIN 
		campaign_contact 
		ON campaign_contact.id = message.campaign_contact_id
	INNER JOIN 
		campaign
		ON campaign.id = campaign_contact.campaign_id
	WHERE 
		campaign.is_started = 'TRUE'
		AND (
			campaign.title LIKE '%Retention%' 
			OR campaign.title LIKE '%RETENTION%')
		AND campaign.organization_id = 5
		AND message.is_from_contact = 'FALSE'
	GROUP BY contact_number
	) 
	AS T
GROUP BY contact_date
```
### Query I made w/Ricky that lists each campaign by campaign and responses
```SQL
SELECT
    campaign.id,
    title,
    interaction_step.question,
    q_value,
    answerz
FROM
    public.campaign
JOIN
    public.interaction_step ON campaign.id = interaction_step.campaign_id
JOIN
    (SELECT
        campaign_id AS camp_id,
        i_s.id AS i_s_id,
        q_r.value AS q_value,
        COUNT (q_r.value) AS answerz
    FROM 
        public.question_response AS q_r
    JOIN 
        public.interaction_step AS i_s ON q_r.interaction_step_id = i_s.id
    GROUP BY campaign_id, i_s.id, q_r.value
    ORDER BY campaign_id, i_s.id, answerz DESC) AS rickyz_table
ON campaign.id = camp_id AND interaction_step.id = i_s_id
```

### Texts Received All Time
```SQL
SELECT 
    COUNT (message.id) AS texts_sent_all_time
FROM
    public.message
JOIN
    campaign_contact ON campaign_contact.id = message.campaign_contact_id
JOIN 
    campaign ON campaign.id = campaign_contact.campaign_id
WHERE
    message.is_from_contact = TRUE AND
    campaign.organization_id = 5
```
### Text Leaderboard w/Formatted Dates
```SQL
select 
	u.id as user_id,
	u.last_name,
	u.first_name,
	u.email,
	u.cell,
	to_char(u.created_at AT time zone 'mst', 'FMDy, FMMon FMDD, YYYY at FMHH:MI PM') as join_date,
	u.alias,
	count (message.id) as Sent_Texts_all_time
from
	public.user AS u,
	public.message
JOIN
	campaign_contact ON campaign_contact.id = message.campaign_contact_id
JOIN campaign ON campaign.id = campaign_contact.campaign_id
WHERE
	message.user_id = u.id AND
  	message.is_from_contact = FALSE AND
	message.send_status = 'DELIVERED' AND
	campaign.organization_id = 5
group by
	u.id
order by
	u.id asc
```

### count answers of a specific answer
```SQL
SELECT
	interaction_step.id as answer_id,
	interaction_step.answer_option as answer,
	COUNT (question_response.id) AS answer_count
FROM
	public.interaction_step,
	public.question_response
WHERE
	question_response.interaction_step_id = interaction_step.id AND
	interaction_step.campaign_id = 37
GROUP BY
	interaction_step.id
ORDER BY
	interaction_step.id asc
```

### Kurt's Opt-Out Query
```SQL
SELECT external_id
FROM campaign_contact
INNER JOIN campaign
ON campaign.id = campaign_contact.campaign_id
LEFT OUTER JOIN opt_out
ON campaign_contact.cell = opt_out.cell
WHERE 
	(campaign.title LIKE '%PERSUASION%' or campaign.title LIKE '%Persuasion%')
	AND campaign.is_started = 'TRUE'
	AND is_opted_out = 'FALSE' 
	AND message_status = 'messaged'
	AND error_code IS NULL
	AND opt_out.id IS NULL
	AND external_id NOT IN (
		SELECT external_id
		FROM campaign_contact
		INNER JOIN campaign
		ON campaign.id = campaign_contact.campaign_id
		WHERE 
			(campaign.title LIKE '%PERSUASION%' OR 
		 	 campaign.title LIKE '%Persuasion%')
			AND campaign.is_started = 'TRUE'
			AND (message_status = 'closed' OR 
				 message_status = 'convo'  OR
				 message_status = 'needsResponse') 
	)
GROUP BY external_id
```

### Sent Texts
```SQL
SELECT 
	COUNT(m.id)
FROM 
	message as m
JOIN
	campaign_contact ON campaign_contact.id = m.campaign_contact_id
JOIN 
	campaign ON campaign.id = campaign_contact.campaign_id
WHERE
	campaign.is_started = 'TRUE'
	AND campaign.organization_id = 5
	AND campaign.id > 21
	AND m.is_from_contact = FALSE
```

### Received Texts
```SQL
SELECT 
	COUNT(m.id)
FROM 
	message as m
JOIN
	campaign_contact ON campaign_contact.id = m.campaign_contact_id
JOIN 
	campaign ON campaign.id = campaign_contact.campaign_id
WHERE
	campaign.is_started = 'TRUE'
	AND campaign.organization_id = 5
	AND campaign.id > 21
	AND m.is_from_contact = TRUE
```

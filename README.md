# Spoke Queries
An updating list for keeping Spoke queries.

- [Campaign Contacts Who Opted-Out w/'STOP'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#campaign-contacts-who-opted-out-wstop)
- [Distinct Cells Where People Opted Out w/'Stop'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#distinct-cells-where-people-opted-out-wstop)
- [Proper Date Formatting](https://github.com/eitanshachor/spokequeries/blob/main/README.md#proper-date-formatting)
- [New Voters Contacted Each Day](https://github.com/eitanshachor/spokequeries/blob/main/README.md#new-voters-contacted-each-day)
- [Sent Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#sent-texts)
- [Received Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#received-texts)
- [Texts Delivered](https://github.com/eitanshachor/spokequeries/blob/main/README.md#texts-delivered)
- [Texts Received Excluding 'Stop'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#texts-received-excluding-stop)
***

## Topline Numbers w/Query Links

| Metric | Number | Context | Updated |
| :--- | ---: | --- | ---: |
| [Sent Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#sent-texts) | `3,203,204` | The number of attempted texts we tried to send to people on our lists. | 12/10/20 |
| [Texts Delivered](https://github.com/eitanshachor/spokequeries/blob/main/README.md#texts-delivered) | `2,794,488` | Out of the number of attempted texts, the number of texts that were actually delivered (as far as we know from our data). | 12/10/20 |
| [Received Texts](https://github.com/eitanshachor/spokequeries/blob/main/README.md#received-texts) | `515,111` |  The number of texts we received throughout the campaign. This metric includes opt-out 'stop' texts. | 12/10/20 |
| [Texts Received Excluding 'Stop'](https://github.com/eitanshachor/spokequeries/blob/main/README.md#texts-received-excluding-stop) | `308,741` | A huge percentage of texts we received were people opting out of future text messages. We had to include opt-out langue later on, so about 2/5 of our total received texts were people not wanting to engage. | 12/10/20 |

### This one calculates lots of important stuff
```SQL
SELECT
    c.id AS camp_id,
    c.title AS title,
    c.organization_id AS org,
    texters::int,
    contacts::int,
    texts_sent::int,
    replies::dec,
    round((replies::decimal / texts_sent), 4)  AS reply_rate,
    meaningfulreplies::dec,
    round((meaningfulreplies::decimal / texts_sent), 4)  AS meaningful_reply_rate,
    meaningfulreplyers::int,
    round((meaningfulreplyers::decimal / contacts), 4) as meaningfulreplyerrate,
    total_texts::dec,
    opt_outs::int,
    round((opt_outs::decimal / texts_sent), 4)  AS opt_out_rate,
    /* cost per texter = total_texts $0.013 assumes typical text is two 'message segments', (one message segment is $0.0065) see https://www.twilio.com/blog/2017/03/what-the-heck-is-a-segment.html#segment  */
    round((total_texts * .013), 4) AS costid,
    costid_2::dec,
    round((costid_2::decimal / contacts), 4) as cost4contact,
    round((costid_2::decimal / NULLIF(replies, 0)), 4) as cost4reply,
    round((costid_2::decimal / NULLIF(meaningfulreplyers, 0)), 4) as cost4goodcontact
FROM public.campaign AS c
INNER JOIN public.organization AS o ON o.id = c.organization_id
INNER JOIN (
    SELECT
        campaign_id,
        COUNT (DISTINCT m.user_id) AS texters,
	COUNT (DISTINCT m.campaign_contact_id) AS contacts,
        COUNT(DISTINCT (
            CASE WHEN m.is_from_contact = 'false' THEN m.id END
            )) AS texts_sent,
        COUNT(DISTINCT (
            CASE WHEN m.is_from_contact = 'true' THEN m.id END
            )) AS replies,
	COUNT(DISTINCT (
            CASE WHEN m.is_from_contact = 'true' 
	    AND m.text != 'STOP'
	    AND m.text != 'stop'
	    AND m.text != 'Stop'
	    THEN m.id END
            )) AS meaningfulreplies,
	COUNT(DISTINCT (
            CASE WHEN m.is_from_contact = 'true' 
	    AND m.text != 'STOP'
	    AND m.text != 'stop'
	    AND m.text != 'Stop'
	    THEN m.campaign_contact_id END
            )) AS meaningfulreplyers,
        COUNT(DISTINCT m.id) AS total_texts,
        COUNT(DISTINCT (
	    CASE WHEN cc.is_opted_out = 'true' THEN cc.id END
	    )) AS opt_outs,
	SUM(CEIL(LENGTH(m.text)::numeric / 160) * .0065) as costid_2
    FROM public.campaign_contact cc
    JOIN public.message m ON m.campaign_contact_id = cc.id
    GROUP BY 1)
AS n ON n.campaign_id = c.id
WHERE campaign_id >= 23 AND
    o.id = 5
ORDER BY 1,2
```


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

### Texts Received Excluding 'Stop'
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
	AND m.send_status = 'DELIVERED'
	AND m.text != 'STOP'
	AND m.text != 'stop'
	AND m.text != 'Stop'
```

### Texts Delivered
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
	AND m.send_status = 'DELIVERED'
```

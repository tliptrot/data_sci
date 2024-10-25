# Algerian Election Data

Part of my dissertation demonstrated a thoery linking water access to party system formation in the Middle East. Regions with limited water tend to practice nomadic pastoralism, which involves moving frequently through the desert with vulnerable cattle herds. To protect themselves, pastoralists tend to build very strong kin networks called tribes or clans encompassing tens or hundreds of thousands of "cousins". Today, these tribes are effective electoral vehicles and "compete" with parties for votes and political space. For example, Saudi Arabia is a "clan" state in the same way the Soviet Union was a "party state". I needed to show that this pattern is observable across different regions within the same state, so I turned to Algeria.

Algeria's 2020 elections saw approximately 20\% of seats go to independent, tribal candidates. I want to show that these independents come mainly from Algeria's arid, pastoralist south and not the wet, farmer-dominated north. However, proving that would require aggregating data from candidates, parties, districts and climate datasets all together. I decided to use this opportunity to show of my skills integrating data in different formats, particularly in SQL and Python.

Objective - For each district, find the proportion of winning candidates who come from small/independent party lists, and merge that with climate data for each district.

## Part 1: A Relational Database of Partisans

First, I need electoral data. Developing countries tend to release data through their "Official Journal", which is a sort of newspaper that the government publishes regularly with important announcements. The base data for candidates looks like this 

![Algerian Official Journal](images/algeria_img_1.jpg)

I transform it into a CSV and massage it into an appropriate format. This snippet creates the Candidates table and uploads from the CSV.

```sql
CREATE TABLE Candidates (
    candidate_id SERIAL PRIMARY KEY,
    last_name VARCHAR(50),
    first_name VARCHAR(50),
	party_list VARCHAR(100),
	party_id INTEGER,
	seats_party_wilaya INTEGER,
	wilaya VARCHAR(50),
	wilaya_id INTEGER,
	seats_wilaya INTEGER
);

COPY Candidates(wilaya_id, wilaya, seats_wilaya, party_list, seats_party_wilaya, last_name, first_name, candidate_id)
FROM 'C:\Program Files\PostgreSQL\17\data\algeria_candidates_csv.csv'
DELIMITER ','
CSV HEADER;
```

Next I use the candidates table to create a parties table.

```sql
CREATE TABLE pl (
    party_id SERIAL PRIMARY KEY,
    party_list VARCHAR(50),
    total_seats_2021 INTEGER,
	major_party BOOLEAN,
	minor_party BOOLEAN,
	independent_list BOOLEAN
	);

INSERT INTO pl (party_list)
SELECT DISTINCT party_list
FROM Candidates;	

UPDATE pl
SET total_seats_2021 = subquery.candidate_count
FROM (
    SELECT party_list, COUNT(*) AS candidate_count
    FROM Candidates
    GROUP BY party_list
) AS subquery
WHERE pl.party_list = subquery.party_list;
```

I  group the parties into three categories based on whether the electeds identified as partisans after entering office. The large parties are those with more than 10 seats, the minor parties have few seats but still call themselves parties, and the "independenets" are a residual category.

```sql
UPDATE pl
SET major_party = CASE
    WHEN total_seats_2021 >= 10 THEN TRUE
    ELSE FALSE
END;

UPDATE pl
SET minor_party = CASE
	WHEN party_list IN ('LA VOIX DU PEUPLE', 'FRONT DE LA BONNE GOUVERNANCE', 'PARTI DE LA LIBERTE ET DE LA JUSTICE
','El FEDJR EL JADID','PARTI DE LA LIBERTE ET DE LA JUSTICE','JIL JADID','FRONT DE L''ALGERIE NOUVELLE
','PARTI EL KARAMA','PARTI EL KARAMA',' FRONT NATIONAL ALGERIEN') THEN TRUE
	ELSE FALSE
END;

UPDATE pl
SET independent_list = CASE
	WHEN minor_party = FALSE AND major_party = FALSE THEN TRUE
	ELSE FALSE
END;

UPDATE Candidates
SET party_id = pl.party_id
FROM pl
WHERE Candidates.party_list = pl.party_list;
```

Next I mark the candidates with these party categories, because candidates can  be linked directly to the districts (wilayat in Arabic), unlike the parties.

```sql
ALTER TABLE Candidates
ADD COLUMN major_party BOOLEAN,
ADD COLUMN minor_party BOOLEAN,
ADD COLUMN independent BOOLEAN;

UPDATE Candidates
SET major_party = pl.major_party
FROM pl
WHERE Candidates.party_id = pl.party_id;

UPDATE Candidates
SET minor_party = pl.minor_party
FROM pl
WHERE Candidates.party_id = pl.party_id;

UPDATE Candidates
SET independent = pl.independent_list
FROM pl
WHERE Candidates.party_id = pl.party_id;

SELECT * FROM Candidates;
```
Finally, I create the table for districts (wilayat) and I take their candidate count by party type.

```sql
CREATE TABLE wl (
    wilaya_id SERIAL PRIMARY KEY,
    wilaya VARCHAR(50),
    total_seats_2021 INTEGER,
	seats_major_party INTEGER,
	seats_independent INTEGER,
	seats_minor_party INTEGER
	);

INSERT INTO wl (wilaya)
SELECT DISTINCT wilaya
FROM Candidates;	

UPDATE wl
SET total_seats_2021 = (
    SELECT COUNT(*)
    FROM Candidates
    WHERE Candidates.wilaya_id = wl.wilaya_id
);

UPDATE wl
SET seats_major_party = (
    SELECT COUNT(*)
    FROM Candidates
    WHERE Candidates.wilaya_id = wl.wilaya_id
    AND Candidates.major_party = TRUE
);

UPDATE wl
SET seats_minor_party = (
    SELECT COUNT(*)
    FROM Candidates
    WHERE Candidates.wilaya_id = wl.wilaya_id
    AND Candidates.minor_party = TRUE
);

UPDATE wl
SET seats_independent = (
    SELECT COUNT(*)
    FROM Candidates
    WHERE Candidates.wilaya_id = wl.wilaya_id
    AND major_party = FALSE
    AND minor_party = FALSE
);

SELECT * FROM wl;
```

## Climate data

Next, I need to 


## Contents

- ### R

     - [How Fast is COVID 19 growing in the US?](https://rpubs.com/tliptrot/596250) The other day my mom asked me "Tim, how quickly is covid actually spreading", and I realized this is actually a difficult question to answer. People are used to thinking about geometric growth, the staight lines of $y = mx + b$. But the behavior of a virus in the early stage of an epidemic is close to exponential growth, $y = e^{ax}$. This demonstration uses some simple graphing and fitting to teach readers about the growth of COVID-19 in the US over the past two months.
     
     - [Inferential Statistics: Does a dietary supplement make people smarter?](https://rpubs.com/tliptrot/581110): A dietary supplement marketer has organized a study to investigate the effects of their product. They had randomly assigned students take different supplements or none, then complete mental math problems as fast as they good. But did the supplements really make people smarter?

     - [Visualizing Survey Data and Linear Regression - Attitudes toward Syrian refugees among rural Jordanians](https://rpubs.com/tliptrot/567264): Uses survey data from humanitarians to vizualize correlations between demographic characteristics and attitudes toward refugees. What subsets of Jordanians have particularly trusting attitudes toward Jordanians?

     - [Data Visualization: The Natural Resource Curse](https://rpubs.com/tliptrot/593873): In this demo, I will make an economist style plot from a multiple country linear regression. For my data I will use country level data from the seminal paper "Natural Resource Abundance and Economic Growth" by Sachs and Warner, which argued that the presence of natural resources in a country caused lower growth.
     
- ### Python

     - [Supervised Learning Project: Predicting Political Attitudes with Developing Country Government Data](https://github.com/tliptrot/data_sci/blob/master/Python/supervised_learning_project:_predictiving_politicla_attitudes.ipynb): This code attempts to create a model which uses administrative data available to authoritarian developing states to predict attitudes toward the authoritarian regime, with supervised learning. The data used comes from a social cohesion survey conducted in Armenia in 2011. Results are inconclusive, and are meant purely as a demonstration of grasp of machine learning concepts.

- ### ArcMap

     - [Refugee Assistance Programming in Northern Jordan](https://github.com/tliptrot/data_sci/blob/master/spatiala_analysis_and_visualization/Reach%20Mafraq%20local%20map_28.pdf): This thematic map was produced for Agency France for Technical Assistance (ACTED) to advertsie the breadth of their programming to prospective funders. They also mounted it on the walls in their field offices, which I was flattered by.
     
- ### Powerpoints

	- [Parents First Advising Sample](https://github.com/casbahboy/Non-Profit-Research/blob/master/Parents%20First%20M%26E%20Advising%20Sample.pptx): This is a powerpoint presentation written for a fictional charity in Cote D'Ivoire that wants to improve their impact on early childhood development. It describes some of the current issues in their monitoring program and advises on practical next steps in targeting data and impact evaluation. It was written as a timed skills-test for Innovations for Poverty Action, so it was completed in just two hours.

  - [Dams of Jordan Social Cohesion and Area Mapping Report](https://github.com/casbahboy/Non-Profit-Research/blob/master/Dams%20of%20Jordan%20Social%20Cohesion%20and%20Area%20Mapping%20Report.pptx) : This powerpoint presentation describes surveys for location targeting in rural Jordan. The survey was commissioned after violence between community members and refugee cash for workers threatened the viability of GIZ's programming. It was given to GIZ and Jordanian Ministry of Water Staff.

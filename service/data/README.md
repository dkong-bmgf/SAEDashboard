# Data Directory Overview

This directory contains the raw data inputs used by the dashboard:

- `data/`: Indicator CSV files for different countries, indicators, and/or subgroups
- `shapefiles/`: Pickled shapefiles used for geographic rendering
- `layer_data.json`: Optional overlays on top of base maps
- `events.csv`: Optional annotated events for timeseries visualization

---
# Data Preparation
## Shapefiles
These are .shp GeoJSONs containing the map data and geometry of the features, and they are serialized into .pickle format (which is helpful to store complex
data structures, like dictionaries, in a way that's more efficient to load and retrieve). It's expected that one shapefile exists per country per admin level. The 
naming convention for these files is as follows, with `l1` indicating a country-level shapefile, `l2` indicating admin1-level, and `l3` indicating admin2-level:
```
{country}__l{hierarchy_level}__{shapefile_version}.shp.pickle
```
To create the shapefiles usable by the backend:
1. Download the country GADM shapefiles needed for your use case from https://gadm.org/
2. You may store the .json files either in `service/data/shapefiles` or in a temporary directory for processing.
3. Run the following script to convert the GADM shapefiles into the pickled shapefiles in the format expected by the backend service:
    ```
    python service/helpers/gadm_geojson_converter.py -i {path to GADM geojsons} -o {path to desired output folder} -c {continent name; "Africa" by default} 
   ```
   For further detail on how this script works, see docs in `service/helpers/gadm_geojson_converter.py`.

>You can include multiple shapefile versions for different layers/indicators on the map; these can be distinguished using 
> the `shapefile_version` parameter in the shapefile name (as indicated in the format above). For example, if I want my 'incidence' 
> indicator for Senegal to use the shapefile version 2 of Senegal at the admin2 level, I can indicate this by having the versions of the indicator 
> filename and shapefile name match as follows: `Senegal__incidence__all__2.csv` and `Senegal__l3__2.shp.pickle`
   
The scripts `serialize_files.py` and `deserialize_files.py` in `service/helpers` can also be used to convert .shp.pickle files to .json files
and vice versa for modification or investigation.

You may also create scripts to both download and process GADM shapefiles into usable shapefiles for the dashboard; see `service/helpers/downloadGeoJson.sh` as an example, which
was created to seamlessly download and process Senegal admin1 and admin2 level shapefiles. 

## Layer Data
To add data layers to the dashboard, create the file `service/layer_data.json` which can contain nested data for layers to display on top of the map in the dashboard. See existing `service/layer_data.json` as an example.
## Event Data
To include events in the timeseries data, create the file `service/events.csv`, which can contain a list of time-bound events relevant to the country context - such as mass gatherings, holidays, or interventions - along with the corresponding date ranges. See existing `service/events.csv` as an example.
   - Each row represents a single event with the following columns:
      - `event`: string
      - `start_date`: string (YYYY-MM-DD)
      - `end_date`: string (YYYY-MM-DD)
## Indicator Data
This section explains how to create and register a new indicator to be displayed in the SenegalMEG dashboard. It includes both **backend** and **frontend** instructions.

### Backend Steps
1. **Prepare the CSV File**  
   - File should be saved in `service/data/data/`
     - Use this naming format:
       ```
       {country}__{indicator_name}__{subgroup}__{shapefile_version}.csv
       ```
       Examples: `Senegal__predicted_incidence__all__1.csv`, `Senegal__CDM__all__2.csv`
     - An indicator can have files for multiple subgroups, i.e. 'all', 'under5', 'over50', etc.
   - **Required columns (for standard indicators):**
            
       | Column             | Type    | Description                                                                                                                                               |
       |--------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
       | `state`            | `str`   | Dotname for the region in format `{continent}:{country}:{admin1}:{admin2}` (i.e. `Africa:Senegal:Tambacounda`, etc.)                                      |
       | `{indicator}`      | `float` | Column for reference data if small area estimates are used in dashboard (values can be blank if not used); MUST match the indicator name used in filename |
       | `se.{indicator}`   | `float` | Column for standard error of reference data if small area estimates used (values blank otherwise)                                                         |
       | `month`            | `int/str`| Optional column to represent month of data. Use `1–12` or `'all'` if displaying an annualized value that shouldn't be summed from monthly values          |
       | `year`             | `int`   | Year of the data                                                                                                                                          |
       | `pred`             | `float` | Primary indicator value containing data for respective subgroup, region, month (if applicable), and year                                                  |
       | `pred_upper`       | `float` | Optional. If blank, defaults to value of `pred` column                                                                                                    |
       | `pred_lower`       | `float` | Optional. If blank, defaults to value of `pred` column                                                                                                    |
   - **Disaggregated Indicator Format (e.g., species composition):** 
     You may want to create an indicator involving multiple values per region per time unit that you want the ability to display in one view on the map (vs. separate subgroups) . One example of this 
       would be the percentage breakdown of mosquito species per region per year. If so, follow this format:
        
       | Column                                                   | Type     | Description                                                                                                                                                                       |
       |----------------------------------------------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
       | `state`, `month`, `year`, `{indicator}`, `se.{indicator}` | see above |
       | `pred`, `pred_upper`, `pred_lower`                       | `float` | Data for the primary group (which is best to include in the indicator name for clarity, e.g., `pred_species_comp_gambiae`). If the data isn't available for this group, put '0' instead of leaving the cell blank. |
       | `pred_{subindicator}`                                    | `float` | Column(s) containing data for additional disaggregated groups of this indicator for the specified state/region and month/year. These must start with `pred_` to be parsed by the backend.                                                                                                                               


2. **Update `config.yaml`**  
- Open `service/config.yaml`
- If you are adding a disaggregated indicator to the dashboard, you MUST add it to the `disaggregated_indicators` section.

---

### Frontend Steps

1. **Update `ConstTs.tsx`**
- File: `client/src/components/ConstTs.tsx`; replace `{indicator_name}` with your indicator
- Add the following json fragment to the end of `IndicatorConfig`:
  ```ts
  '{indicator_name}': {'unitLabel': '{indicator_name}', 
                    'multiper': 1, 
                    'unit': '', 
                    'mapLabel': '', 
                    'legendLabel': '{indicator_name}', 
                    'decimatlPt': 0, 
                    'extraInfo': false},
  ```

2. **Update Indicator Dropdown**
- File: `client/src/components/filterelements/IndicatorFilter.js` (this file is used to store indicator options in the Indicator dropdown); replace `{indicator_name}` with your indicator
- Add a new `<MenuItem>` as a child component in the `Select` component:
  ```jsx
  <MenuItem value="{indicator_name}" key={15} className={classes.menuItem}>
    <FormattedMessage id="{indicator_name}"/>
  </MenuItem>
  ```

---

💡 After adding the files and config:
- Add the project root path to PYTHONPATH environment variable (if not done so already) and close and reopen your IDE:
   - on Windows (CMD):
        ```bash 
        set PYTHONPATH=%PYTHONPATH%;C:\Your\Project\Root
        set ENV=development
        ```
   - on macOS/Linux:
     ```bash 
     export PYTHONPATH=$PYTHONPATH:$(pwd)
     export ENV=development
     ```
- Restart the backend (`python app.py manage run`)
- Restart the frontend (`yarn start`)

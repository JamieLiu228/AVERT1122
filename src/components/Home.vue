<template>
  <div class="home">
    <b-img :src="require('../assets/avert_logo.png')" fluid alt="Avert Logo"></b-img>
    <!-- patient id dropdown menu which load all patient id when page is loaded completely via calling getAllPatientIds function -->
    <b-container fluid="sm" class="text-left">
        <h3>Patient</h3>
        <label for="patient-id-select">Patient ID: </label>
        <b-form-select id="patient-id-select" v-model="selected_patient_id" :options="patient_ids"></b-form-select>
    </b-container>
    <br />
    <!-- return type checkbox for use to select which return type should return -->
    <b-container fluid="sm" class="text-left">
        <h3>Return Type</h3>
    </b-container>
    <b-container fluid="sm" class="text-left">
      <b-form-group>
        <b-form-radio-group v-model="selected_return_type">
          <b-form-radio name="pollution" value="pollution">Pollution</b-form-radio>
          <b-form-radio name="weather" value="weather">Weather</b-form-radio>
          <b-form-radio name="all" value="all">All</b-form-radio>
        </b-form-radio-group>
      </b-form-group>
    </b-container>
    <br />
    <!-- from and to date selction module -->
    <b-container fluid="sm" class="text-left">
        <h3>Date</h3>
    </b-container>
    <b-container fluid="sm" class="text-left">
      <b-form>
        <label for="from-date-datepicker">From: </label>
        <b-form-datepicker id="from-date-datepicker" v-model="selected_from_date" class="mb-2"></b-form-datepicker>
        <label for="to-date-datepicker">To: </label>
        <b-form-datepicker id="to-date-datepicker" v-model="selected_to_date" class="mb-2"></b-form-datepicker>
      </b-form>
    </b-container>
    <br />
    <b-container fluid="sm">
      <!-- submit button calling submitQuery function to send requests -->
      <b-button class="ml-4" style="width: 100px;" variant="outline-primary" @click="submitQuery()">Search</b-button>
      <!-- download button calling download_csv function to export csv file -->
      <b-button class="ml-4" style="width: 100px;" variant="outline-primary" @click="download_csv()">Downoad</b-button>
      <!-- Feedback button to show feedback modal for use to write a comment -->
      <b-button class="ml-4" style="width: 100px;" variant="outline-primary" @click="$refs['feedback-modal'].show()">Feedback</b-button>
    </b-container>
    <br />
    <b-container fluid="sm">
        <h3 v-if="this.selected_return_type == 'pollution'">Pollution Results</h3>
        <h3 v-if="this.selected_return_type == 'weather'">Weather Results</h3>
        <h3 v-if="this.selected_return_type == 'all'">All Results</h3>
        <b-spinner v-if="this.loading == true" variant="primary" label="Spinning"></b-spinner>
        <br />
        <span v-if="this.loading == false">Execution Time: {{ this.execution_time  }} sec </span>
    </b-container>
    <br />

    <b-container fluid>
        <b-table striped hover :items="this.table"></b-table>
    </b-container>
    
    <b-modal ref="feedback-modal" hide-footer title="Feedback">
      <div class="d-block text-center">
        <h3>Please write your comment here</h3>
        <textarea class="form-control" id="exampleFormControlTextarea1" rows="3"></textarea>
      </div>
      <b-button class="mt-3" variant="outline-primary" block @click="$refs['feedback-modal'].hide()">Submit</b-button>
      <b-button class="mt-2" variant="outline-danger" block @click="$refs['feedback-modal'].hide()">Cancle</b-button>
    </b-modal>
  </div>
</template>

<script>
const { ServerClient, ServerClientConfig } = require('graphdb').server;
const { RDFMimeType } = require('graphdb').http;
const { RepositoryClientConfig } = require('graphdb').repository;
const { GetQueryPayload, QueryType } = require('graphdb').query;
const { SparqlJsonResultParser } = require('graphdb').parser;

const readTimeout = 600000;
const writeTimeout = 600000;

const server_config = new ServerClientConfig('http://localhost:7200/', 0, {
    'Accept': RDFMimeType.SPARQL_RESULTS_JSON
});
const repository_config = new RepositoryClientConfig(['http://localhost:7200/repositories/test'], {
  'Accept': RDFMimeType.TURTLE
}, '', readTimeout, writeTimeout);
const server = new ServerClient(server_config);

export default {
  name: 'Home',
  data() {
    return {
      patient_ids: [],
      selected_patient_id: null,
      selected_return_type: 'pollution',
      selected_from_date: '2018-01-01',
      selected_to_date: '2018-01-31',
      surrounding_pollution_locations: [],
      threshold: 3,
      results: {},
      table: [],
      loading: false,
      execution_time: 0
    }
  },
  methods: {
    // generate and export CSV for downlaoding
     download_csv() {
         let csv = '';
         Object.keys(this.table[0]).forEach((key) =>{
             csv += key
             csv += ','
         })
         csv += '\n';
         this.table.forEach((row) =>{
             Object.keys(row).forEach((key) => {
                csv += String(row[key])
                csv += ',';
            })
             csv += '\n';
          });
         console.log(csv);
         const hiddenElement = document.createElement('a');
         hiddenElement.href = 'data:text/csv;charset=utf-8,' + encodeURI(csv);
         hiddenElement.target = '_blank';
         hiddenElement.download = this.selected_return_type + '_' + this.selected_from_date + '_' + this.selected_to_date+'.csv';
         hiddenElement.click();
     },
    // list all patient ids
    getAllPatientIds() {
      // get test repository from graphdb
      server.getRepository('test', repository_config).then((repository) => {
        // register JSON result parse as return type from graphdb
        repository.registerParser(new SparqlJsonResultParser());
        // polution query as playload in request
        const payload = new GetQueryPayload()
          .setQuery('prefix avertd: <http://data.avert.ie/vocabulary/distiller/> SELECT distinct ?pr_pid WHERE { ?pr a <http://data.avert.ie/vocabulary/distiller/DistillerPatientRecord>. ?pr avertd:patientID ?pr_pid.}')
          .setQueryType(QueryType.SELECT)
          .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
          .setLimit(100);
        // send request to graphdb and retrieve results from graphdb
        return repository.query(payload).then((stream) => {
          this.patient_ids = stream.map((item) => { return item.pr_pid.value.replace('ID_', '') })
          this.patient_ids.push('All')
          console.log(this.patient_ids)
        });
      }).catch((error) => console.log(error));
    },
    // logic is same as getAllPatientIds function
    getAllPMPRecordsByPatientId() {
        server.getRepository('test', repository_config).then((repository) => {
        repository.registerParser(new SparqlJsonResultParser());
        
        const payload = new GetQueryPayload()
          .setQuery('PREFIX geo: <http://www.opengis.net/ont/geosparql#> PREFIX xsd: <http://www.w3.org/2001/XMLSchema#> prefix avert: <http://data.avert.ie/vocabulary/> SELECT distinct ?pmpr_pid ?pmpr_date ?pmpr_point WHERE { ?pmpr a avert:PMPrecord. ?pmpr avert:patient_id ?pmpr_pid. ?pmpr avert:time_date ?pmpr_date. ?pmpr geo:hasGeometry ?pmpr_point. FILTER(?pmpr_pid = "pmp_'+ this.selected_patient_id + '" && ?pmpr_date <= "' + this.selected_to_date + '"^^xsd:date && ?pmpr_date >= "' + this.selected_from_date + '"^^xsd:date) }'
          )
          .setQueryType(QueryType.SELECT)
          .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
          .setLimit(100);
        
        return repository.query(payload).then((stream) => {
          console.log(stream)
        });
      }).catch((error) => console.log(error));
    },
    // logic is same as getAllPatientIds function
    async getClosestPollutionRecordsByPMPRecords() {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>

        SELECT ?pmpr_pid ?pmpr_point ?plr_point (?plr_date as ?date) ?min_distance ?plr_oxn_m2grid ?plr_rnd_m2grid ?plr_sox_m2grid ?plr_max03 ?plr_pm25water ?plr_ppb_O3 ?plr_ugn_nox ?plr_ugn_rdn ?plr_ug_dust ?plr_ug_eccoarse ?plr_ug_ecfine ?plr_ug_nh3 ?plr_ug_nh4_f ?plr_ug_no2 ?plr_ug_no3 ?plr_ug_pm10 ?plr_ug_pm25 ?plr_ug_pmco ?plr_ug_ppm_c ?plr_ug_so2 ?plr_ug_so4 ?plr_ug_ss ?plr_wdep_oxn ?plr_wdep_prec ?plr_wdep_rdn ?plr_wdep_sox
        WHERE {
            ?pmpr a avert:PMPrecord.
            ?plr a avert:PollutionRecord.
            {
                SELECT ?pmpr_pid ?pmpr_date ?pmpr_point (MIN(geof:distance(?pmpr_point, ?plr_point, uom:metre)) AS ?min_distance)
                WHERE {
                    ?pmpr avert:patient_id ?pmpr_pid.
                    ?pmpr avert:time_date ?pmpr_date.
                    ?pmpr geo:hasGeometry ?pmpr_point.
                    FILTER(?pmpr_pid = "pmp_${this.selected_patient_id}" && ?pmpr_date >= "${this.selected_from_date}"^^xsd:date && ?pmpr_date <= "${this.selected_to_date}"^^xsd:date)
                    ?plr a avert:PollutionRecord.
                    ?plr geo:hasGeometry ?plr_point.
                    ?plr <http://purl.org/dc/terms/date> ?plr_date
                    FILTER(?plr_date = ?pmpr_date)
                }
                GROUP BY ?pmpr_pid ?pmpr_point ?pmpr_date
            }
            ?plr geo:hasGeometry ?plr_point.
            ?plr <http://purl.org/dc/terms/date> ?plr_date
            FILTER(?plr_date = ?pmpr_date && geof:distance(?pmpr_point, ?plr_point, uom:metre) = ?min_distance)
            ?plr avert:DDEP_OXN_M2GRID_value ?plr_oxn_m2grid.
            ?plr avert:DDEP_RDN_M2GRID_value ?plr_rnd_m2grid.
            ?plr avert:DDEP_SOX_M2GRID_value ?plr_sox_m2grid.
            ?plr avert:SURF_MAXO3_value ?plr_max03.
            ?plr avert:SURF_PM25WATER_value ?plr_pm25water.
            ?plr avert:SURF_PPB_O3_value ?plr_ppb_O3.
            ?plr avert:SURF_UGN_NOX_value ?plr_ugn_nox.
            ?plr avert:SURF_UGN_RDN_value ?plr_ugn_rdn.
            ?plr avert:SURF_UG_DUST_value ?plr_ug_dust.
            ?plr avert:SURF_UG_ECCOARSE_value ?plr_ug_eccoarse.
            ?plr avert:SURF_UG_ECFINE_value ?plr_ug_ecfine.
            ?plr avert:SURF_UG_NH3_value ?plr_ug_nh3.
            ?plr avert:SURF_UG_NH4_F_value ?plr_ug_nh4_f.
            ?plr avert:SURF_UG_NO2_value ?plr_ug_no2.
            ?plr avert:SURF_UG_NO3_C_value ?plr_ug_no3.
            ?plr avert:SURF_UG_PM10_value ?plr_ug_pm10.
            ?plr avert:SURF_UG_PM25_value ?plr_ug_pm25.
            ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
            ?plr avert:SURF_UG_PPM25_value ?plr_ug_ppm25.
            ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
            ?plr avert:SURF_UG_PPM_C_value ?plr_ug_ppm_c.
            ?plr avert:SURF_UG_SO2_value ?plr_ug_so2.
            ?plr avert:SURF_UG_SO4_value ?plr_ug_so4.
            ?plr avert:SURF_UG_SS_value ?plr_ug_ss.
            ?plr avert:WDEP_OXN_value ?plr_wdep_oxn.
            ?plr avert:WDEP_PREC_value ?plr_wdep_prec.
            ?plr avert:WDEP_RDN_value ?plr_wdep_rdn.
            ?plr avert:WDEP_SOX_value ?plr_wdep_sox.
        }
      `
      console.log(query)
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results,'PMP')
    },
    // logic is same as getAllPatientIds function
    async getClosestWeatherRecordsByPMPRecords () {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>

        SELECT ?pmpr_pid ?pmpr_point (?wr_date AS ?date) ?wr_point ?min_distance ?wr_blh ?wr_d2m ?wr_e ?wr_fg10 ?wr_mcc ?wr_mn2t ?wr_msl ?wr_par ?wr_ro ?wr_sf ?wr_skt ?wr_slhf ?wr_sp ?wr_ssr ?wr_sst ?wr_stl1 ?wr_sund ?wr_swvl1 ?wr_t2m ?wr_tcc ?wr_tco3 ?wr_tcwv ?wr_tcw ?wr_tp ?wr_u10 ?wr_v10
        WHERE {
            ?pmpr a avert:PMPrecord.
            ?wr a avert:WeatherRecord.
            {
                SELECT ?pmpr_pid ?pmpr_date ?pmpr_point (MIN(geof:distance(?pmpr_point_0, ?wr_point_0, uom:metre)) AS ?min_distance)
                WHERE {
                    ?pmpr avert:patient_id ?pmpr_pid.
                    ?pmpr avert:time_date ?pmpr_date.
                    ?pmpr geo:hasGeometry ?pmpr_point.
                    FILTER(?pmpr_pid = "pmp_${this.selected_patient_id}" && ?pmpr_date >= "${this.selected_from_date}"^^xsd:date && ?pmpr_date <= "${this.selected_to_date}"^^xsd:date)
                    ?wr geo:hasGeometry ?wr_point.
                    ?wr <http://purl.org/dc/terms/date> ?wr_date.
                    FILTER(?wr_date = ?pmpr_date)
                }
                GROUP BY ?pmpr_pid ?pmpr_point ?pmpr_date
            }
            ?wr geo:hasGeometry ?wr_point.
            ?wr <http://purl.org/dc/terms/date> ?wr_date
            FILTER(?wr_date = ?pmpr_date && geof:distance(?pmpr_point, ?wr_point, uom:metre) = ?min_distance)
            ?wr avert:BLH_value ?wr_blh.
            ?wr avert:D2M_value ?wr_d2m.
            ?wr avert:E_value ?wr_e.
            ?wr avert:FG10_value ?wr_fg10.
            ?wr avert:MCC_value ?wr_mcc.
            ?wr avert:MN2T_value ?wr_mn2t.
            ?wr avert:MSL_value ?wr_msl.
            ?wr avert:MX2T_value ?wr_mx2t.
            ?wr avert:PAR_value ?wr_par.
            ?wr avert:RO_value ?wr_ro.
            ?wr avert:SF_value ?wr_sf.
            ?wr avert:SKT_value ?wr_skt.
            ?wr avert:SLHF_value ?wr_slhf.
            ?wr avert:SP_value ?wr_sp.
            ?wr avert:SSHF_value ?wr_sshf.
            ?wr avert:SSR_value ?wr_ssr.
            ?wr avert:SST_value ?wr_sst.
            ?wr avert:STL1_value ?wr_stl1.
            ?wr avert:SUND_value ?wr_sund.
            ?wr avert:SWVL1_value ?wr_swvl1.
            ?wr avert:T2M_value ?wr_t2m.
            ?wr avert:TCC_value ?wr_tcc.
            ?wr avert:TCO3_value ?wr_tco3.
            ?wr avert:TCWV_value ?wr_tcwv.
            ?wr avert:TCW_value ?wr_tcw.
            ?wr avert:TP_value ?wr_tp.
            ?wr avert:U10_value ?wr_u10.
            ?wr avert:V10_value ?wr_v10.
        }
      `;
      console.log(query);
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results,'PMP')
    },
    // logic is same as getAllPatientIds function
    async getClosestPollutionRecordsByPatientEDRecord() {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>

        SELECT ?plr ?plr_point (?plr_date AS ?date) ?plr_oxn_m2grid ?plr_rnd_m2grid ?plr_sox_m2grid ?plr_max03 ?plr_pm25water ?plr_ppb_O3 ?plr_ugn_nox ?plr_ugn_rdn ?plr_ug_dust ?plr_ug_eccoarse ?plr_ug_ecfine ?plr_ug_nh3 ?plr_ug_nh4_f ?plr_ug_no2 ?plr_ug_no3 ?plr_ug_pm10 ?plr_ug_pm25 ?plr_ug_pmco ?plr_ug_ppm_c ?plr_ug_so2 ?plr_ug_so4 ?plr_ug_ss ?plr_wdep_oxn ?plr_wdep_prec ?plr_wdep_rdn ?plr_wdep_sox
        WHERE {
          ?pel a avert:PatientEDLocation.
          
          ?pel avert:edRegion ?edr.
          ?pel avert:patientID ?pel_pid.
          ?edr <http://ontologies.geohive.ie/osi#partOf> ?county.
          
          FILTER(?pel_pid = "ID_${this.selected_patient_id}")
          
          ?plr a avert:PollutionRecord.
          ?plr geo:hasGeometry ?plr_point.
          ?plr <http://purl.org/dc/terms/date> ?plr_date.
          
          ?county <http://www.opengis.net/ont/geosparql#hasGeometry> ?county_geo.
          ?county_geo geo:asWKT ?county_polygon.
          FILTER(geof:sfWithin(?plr_point, ?county_polygon) && ?plr_date >= "${this.selected_from_date}"^^xsd:date && ?plr_date <= "${this.selected_to_date}"^^xsd:date).

          ?plr avert:DDEP_OXN_M2GRID_value ?plr_oxn_m2grid.
          ?plr avert:DDEP_RDN_M2GRID_value ?plr_rnd_m2grid.
          ?plr avert:DDEP_SOX_M2GRID_value ?plr_sox_m2grid.
          ?plr avert:SURF_MAXO3_value ?plr_max03.
          ?plr avert:SURF_PM25WATER_value ?plr_pm25water.
          ?plr avert:SURF_PPB_O3_value ?plr_ppb_O3.
          ?plr avert:SURF_UGN_NOX_value ?plr_ugn_nox.
          ?plr avert:SURF_UGN_RDN_value ?plr_ugn_rdn.
          ?plr avert:SURF_UG_DUST_value ?plr_ug_dust.
          ?plr avert:SURF_UG_ECCOARSE_value ?plr_ug_eccoarse.
          ?plr avert:SURF_UG_ECFINE_value ?plr_ug_ecfine.
          ?plr avert:SURF_UG_NH3_value ?plr_ug_nh3.
          ?plr avert:SURF_UG_NH4_F_value ?plr_ug_nh4_f.
          ?plr avert:SURF_UG_NO2_value ?plr_ug_no2.
          ?plr avert:SURF_UG_NO3_C_value ?plr_ug_no3.
          ?plr avert:SURF_UG_PM10_value ?plr_ug_pm10.
          ?plr avert:SURF_UG_PM25_value ?plr_ug_pm25.
          ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
          ?plr avert:SURF_UG_PPM25_value ?plr_ug_ppm25.
          ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
          ?plr avert:SURF_UG_PPM_C_value ?plr_ug_ppm_c.
          ?plr avert:SURF_UG_SO2_value ?plr_ug_so2.
          ?plr avert:SURF_UG_SO4_value ?plr_ug_so4.
          ?plr avert:SURF_UG_SS_value ?plr_ug_ss.
          ?plr avert:WDEP_OXN_value ?plr_wdep_oxn.
          ?plr avert:WDEP_PREC_value ?plr_wdep_prec.
          ?plr avert:WDEP_RDN_value ?plr_wdep_rdn.
          ?plr avert:WDEP_SOX_value ?plr_wdep_sox.
        }
      `;
      console.log(query);
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results,'ED')
    },
    // logic is same as getAllPatientIds function
    async getClosestWeatherRecordsByPatientEDRecord() {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>

        SELECT ?wr ?wr_point (?wr_date AS ?date) ?wr_blh ?wr_d2m ?wr_e ?wr_fg10 ?wr_mcc ?wr_mn2t ?wr_msl ?wr_par ?wr_ro ?wr_sf ?wr_skt ?wr_slhf ?wr_sp ?wr_ssr ?wr_sst ?wr_stl1 ?wr_sund ?wr_swvl1 ?wr_t2m ?wr_tcc ?wr_tco3 ?wr_tcwv ?wr_tcw ?wr_tp ?wr_u10 ?wr_v10
        WHERE {
          ?pel a avert:PatientEDLocation.
          
          ?pel avert:edRegion ?edr.
          ?pel avert:patientID ?pel_pid.
          ?edr <http://ontologies.geohive.ie/osi#partOf> ?county.
          FILTER(?pel_pid = "ID_${this.selected_patient_id}")
              
          ?wr a avert:WeatherRecord.
          ?wr geo:hasGeometry ?wr_point.
          ?wr <http://purl.org/dc/terms/date> ?wr_date.
          
          ?county <http://www.opengis.net/ont/geosparql#hasGeometry> ?county_geo.
          ?county_geo geo:asWKT ?county_polygon.
          FILTER(geof:sfWithin(?wr_point, ?county_polygon) && ?wr_date >= "${this.selected_from_date}"^^xsd:date && ?wr_date <= "${this.selected_to_date}"^^xsd:date).

          ?wr avert:BLH_value ?wr_blh.
          ?wr avert:D2M_value ?wr_d2m.
          ?wr avert:E_value ?wr_e.
          ?wr avert:FG10_value ?wr_fg10.
          ?wr avert:MCC_value ?wr_mcc.
          ?wr avert:MN2T_value ?wr_mn2t.
          ?wr avert:MSL_value ?wr_msl.
          ?wr avert:MX2T_value ?wr_mx2t.
          ?wr avert:PAR_value ?wr_par.
          ?wr avert:RO_value ?wr_ro.
          ?wr avert:SF_value ?wr_sf.
          ?wr avert:SKT_value ?wr_skt.
          ?wr avert:SLHF_value ?wr_slhf.
          ?wr avert:SP_value ?wr_sp.
          ?wr avert:SSHF_value ?wr_sshf.
          ?wr avert:SSR_value ?wr_ssr.
          ?wr avert:SST_value ?wr_sst.
          ?wr avert:STL1_value ?wr_stl1.
          ?wr avert:SUND_value ?wr_sund.
          ?wr avert:SWVL1_value ?wr_swvl1.
          ?wr avert:T2M_value ?wr_t2m.
          ?wr avert:TCC_value ?wr_tcc.
          ?wr avert:TCO3_value ?wr_tco3.
          ?wr avert:TCWV_value ?wr_tcwv.
          ?wr avert:TCW_value ?wr_tcw.
          ?wr avert:TP_value ?wr_tp.
          ?wr avert:U10_value ?wr_u10.
          ?wr avert:V10_value ?wr_v10.
        }
      `;
      console.log(query);
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results,'ED')
    },
    // logic is same as getAllPatientIds function
    async getClosestPollutionRecordsByHospitalLocation() {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>
        prefix avertd: <http://data.avert.ie/vocabulary/distiller/>

        SELECT ?hr_name ?hr_point ?min_distance (?plr_date AS ?date) ?plr_oxn_m2grid ?plr_rnd_m2grid ?plr_sox_m2grid ?plr_max03 ?plr_pm25water ?plr_ppb_O3 ?plr_ugn_nox ?plr_ugn_rdn ?plr_ug_dust ?plr_ug_eccoarse ?plr_ug_ecfine ?plr_ug_nh3 ?plr_ug_nh4_f ?plr_ug_no2 ?plr_ug_no3 ?plr_ug_pm10 ?plr_ug_pm25 ?plr_ug_pmco ?plr_ug_ppm_c ?plr_ug_so2 ?plr_ug_so4 ?plr_ug_ss ?plr_wdep_oxn ?plr_wdep_prec ?plr_wdep_rdn ?plr_wdep_sox
        WHERE {
          ?hr a avert:HospitalLocation.
          ?plr a avert:PollutionRecord.
          {
            SELECT ?hr_name ?hr_point (MIN(geof:distance(?hr_point, ?plr_point, uom:metre)) AS ?min_distance)
            WHERE {
              ?pr a avertd:DistillerPatientRecord.
              ?pr avertd:patientID ?pr_pid.
              ?pr avertd:atHospital ?pr_h_name.
              FILTER(?pr_pid = "ID_${this.selected_patient_id}")
              
              ?hr avert:hospitalName ?hr_name.
              ?hr geo:asWKT ?hr_point.
              FILTER(?hr_name = ?pr_h_name)
              
              ?plr a avert:PollutionRecord.
              ?plr geo:hasGeometry ?plr_point.
              ?plr <http://purl.org/dc/terms/date> ?plr_date.
              FILTER(?plr_date >= "${this.selected_from_date}"^^xsd:date && ?plr_date <= "${this.selected_to_date}"^^xsd:date)
            }
            GROUP BY ?hr_name ?hr_point
          }
          ?plr geo:hasGeometry ?plr_point.
          ?plr <http://purl.org/dc/terms/date> ?plr_date
          FILTER(geof:distance(?hr_point, ?plr_point, uom:metre) = ?min_distance && ?plr_date >= "${this.selected_from_date}"^^xsd:date && ?plr_date <= "${this.selected_to_date}"^^xsd:date)
          ?plr avert:DDEP_OXN_M2GRID_value ?plr_oxn_m2grid.
          ?plr avert:DDEP_RDN_M2GRID_value ?plr_rnd_m2grid.
          ?plr avert:DDEP_SOX_M2GRID_value ?plr_sox_m2grid.
          ?plr avert:SURF_MAXO3_value ?plr_max03.
          ?plr avert:SURF_PM25WATER_value ?plr_pm25water.
          ?plr avert:SURF_PPB_O3_value ?plr_ppb_O3.
          ?plr avert:SURF_UGN_NOX_value ?plr_ugn_nox.
          ?plr avert:SURF_UGN_RDN_value ?plr_ugn_rdn.
          ?plr avert:SURF_UG_DUST_value ?plr_ug_dust.
          ?plr avert:SURF_UG_ECCOARSE_value ?plr_ug_eccoarse.
          ?plr avert:SURF_UG_ECFINE_value ?plr_ug_ecfine.
          ?plr avert:SURF_UG_NH3_value ?plr_ug_nh3.
          ?plr avert:SURF_UG_NH4_F_value ?plr_ug_nh4_f.
          ?plr avert:SURF_UG_NO2_value ?plr_ug_no2.
          ?plr avert:SURF_UG_NO3_C_value ?plr_ug_no3.
          ?plr avert:SURF_UG_PM10_value ?plr_ug_pm10.
          ?plr avert:SURF_UG_PM25_value ?plr_ug_pm25.
          ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
          ?plr avert:SURF_UG_PPM25_value ?plr_ug_ppm25.
          ?plr avert:SURF_UG_PMCO_value ?plr_ug_pmco.
          ?plr avert:SURF_UG_PPM_C_value ?plr_ug_ppm_c.
          ?plr avert:SURF_UG_SO2_value ?plr_ug_so2.
          ?plr avert:SURF_UG_SO4_value ?plr_ug_so4.
          ?plr avert:SURF_UG_SS_value ?plr_ug_ss.
          ?plr avert:WDEP_OXN_value ?plr_wdep_oxn.
          ?plr avert:WDEP_PREC_value ?plr_wdep_prec.
          ?plr avert:WDEP_RDN_value ?plr_wdep_rdn.
          ?plr avert:WDEP_SOX_value ?plr_wdep_sox.
        }
      `;
      console.log(query);
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results, 'HL')
    },
    // logic is same as getAllPatientIds function
    async getClosestWeatherRecordsByHospitalLocation() {
      const query = `
        PREFIX geo: <http://www.opengis.net/ont/geosparql#>
        PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
        PREFIX avert: <http://data.avert.ie/vocabulary/> 
        PREFIX geof: <http://www.opengis.net/def/function/geosparql/>
        PREFIX uom: <http://www.opengis.net/def/uom/OGC/1.0/>
        prefix avertd: <http://data.avert.ie/vocabulary/distiller/>

        SELECT ?hr_name ?hr_point ?min_distance (?wr_date AS ?date) ?wr_blh ?wr_d2m ?wr_e ?wr_fg10 ?wr_mcc ?wr_mn2t ?wr_msl ?wr_par ?wr_ro ?wr_sf ?wr_skt ?wr_slhf ?wr_sp ?wr_ssr ?wr_sst ?wr_stl1 ?wr_sund ?wr_swvl1 ?wr_t2m ?wr_tcc ?wr_tco3 ?wr_tcwv ?wr_tcw ?wr_tp ?wr_u10 ?wr_v10
        WHERE {
          ?hr a avert:HospitalLocation.
          ?wr a avert:WeatherRecord.
          {
            SELECT ?hr_name ?hr_point (MIN(geof:distance(?hr_point, ?wr_point, uom:metre)) AS ?min_distance)
            WHERE {
              ?pr a avertd:DistillerPatientRecord.
              ?pr avertd:patientID ?pr_pid.
              ?pr avertd:atHospital ?pr_h_name.
              FILTER(?pr_pid = "ID_${this.selected_patient_id}")
              
              ?hr avert:hospitalName ?hr_name.
              ?hr geo:asWKT ?hr_point.
              FILTER(?hr_name = ?pr_h_name)
              
              ?wr geo:hasGeometry ?wr_point.
              ?wr <http://purl.org/dc/terms/date> ?wr_date.
              FILTER(?wr_date >= "${this.selected_from_date}"^^xsd:date && ?wr_date <= "${this.selected_to_date}"^^xsd:date)
            }
            GROUP BY ?hr_name ?hr_point
          }
          ?wr geo:hasGeometry ?wr_point.
          ?wr <http://purl.org/dc/terms/date> ?wr_date.
          FILTER(geof:distance(?hr_point, ?wr_point, uom:metre) = ?min_distance && ?wr_date >= "${this.selected_from_date}"^^xsd:date && ?wr_date <= "${this.selected_to_date}"^^xsd:date)
          ?wr avert:BLH_value ?wr_blh.
          ?wr avert:D2M_value ?wr_d2m.
          ?wr avert:E_value ?wr_e.
          ?wr avert:FG10_value ?wr_fg10.
          ?wr avert:MCC_value ?wr_mcc.
          ?wr avert:MN2T_value ?wr_mn2t.
          ?wr avert:MSL_value ?wr_msl.
          ?wr avert:MX2T_value ?wr_mx2t.
          ?wr avert:PAR_value ?wr_par.
          ?wr avert:RO_value ?wr_ro.
          ?wr avert:SF_value ?wr_sf.
          ?wr avert:SKT_value ?wr_skt.
          ?wr avert:SLHF_value ?wr_slhf.
          ?wr avert:SP_value ?wr_sp.
          ?wr avert:SSHF_value ?wr_sshf.
          ?wr avert:SSR_value ?wr_ssr.
          ?wr avert:SST_value ?wr_sst.
          ?wr avert:STL1_value ?wr_stl1.
          ?wr avert:SUND_value ?wr_sund.
          ?wr avert:SWVL1_value ?wr_swvl1.
          ?wr avert:T2M_value ?wr_t2m.
          ?wr avert:TCC_value ?wr_tcc.
          ?wr avert:TCO3_value ?wr_tco3.
          ?wr avert:TCWV_value ?wr_tcwv.
          ?wr avert:TCW_value ?wr_tcw.
          ?wr avert:TP_value ?wr_tp.
          ?wr avert:U10_value ?wr_u10.
          ?wr avert:V10_value ?wr_v10.
        }
      `;
      console.log(query);
      let repository = await server.getRepository('test', repository_config);
      repository.registerParser(new SparqlJsonResultParser());
        
      const payload = new GetQueryPayload()
        .setQuery(query)
        .setQueryType(QueryType.SELECT)
        .setResponseType(RDFMimeType.SPARQL_RESULTS_JSON)
        .setLimit(100);

      let results = await repository.query(payload);
      this.parseResults(results, 'HL')
    },
    // parse return results, convert to the dictionry date format and be ready to present in the table (overwrite the results from different performance by priority order)
    parseResults(stream, queryName) {
      stream.forEach((row) => {
        let date = row['date']['value']
        let dict = {}
        dict['patientId'] = this.selected_patient_id
        dict['LocationPreferences'] = queryName
        Object.keys(row).forEach((key) => {
            dict[key] = row[key]['value']
        })
        if(date in this.results) {
            this.results[date] = Object.assign({}, this.results[date], dict);
          } else {
            this.results[date] = dict
        }
      })
    },
    // send query
    async submitQuery() {
      if(this.selected_patient_id == null) {
        alert('Please select patient ID.')
      } else {
        this.results = {}
        this.table = []
        this.loading = true
        let start_at = new Date().getTime();
        // if return type is pollution
        if(this.selected_return_type == 'pollution') {
          // 1
          await this.getClosestPollutionRecordsByHospitalLocation()
          this.table = Object.values(this.results)
          // 2
          await this.getClosestPollutionRecordsByPatientEDRecord()
          this.table = Object.values(this.results)
          // 3
          await this.getClosestPollutionRecordsByPMPRecords()
          this.table = Object.values(this.results)

          this.loading = false
          let end_at = new Date().getTime()
          this.execution_time = (end_at - start_at) / 1000
        // if return type is pollution
        } else if(this.selected_return_type == 'weather') {
          // 1
          await this.getClosestWeatherRecordsByHospitalLocation()
          this.table = Object.values(this.results)
          // 2
          await this.getClosestWeatherRecordsByPatientEDRecord()
          this.table = Object.values(this.results)
          // 3
          await this.getClosestWeatherRecordsByPMPRecords()
          this.table = Object.values(this.results)

          this.loading = false
          let end_at = new Date().getTime()
          this.execution_time = (end_at - start_at) / 1000
        // if return type is all
        } else if(this.selected_return_type == 'all') {
          // 1
          await this.getClosestPollutionRecordsByHospitalLocation()
          await this.getClosestWeatherRecordsByHospitalLocation()
          this.table = Object.values(this.results)
          // 2
          await this.getClosestPollutionRecordsByPatientEDRecord()
          await this.getClosestWeatherRecordsByPatientEDRecord()
          this.table = Object.values(this.results)
          // 3
          await this.getClosestPollutionRecordsByPMPRecords()
          await this.getClosestWeatherRecordsByPMPRecords()
          this.table = Object.values(this.results)

          this.loading = false
          let end_at = new Date().getTime()
          this.execution_time = (end_at - start_at) / 1000
        }
      }
    }
  },
  created() {
    // execute get all patient ids when page is created
    this.getAllPatientIds()
  }
}
</script>

<style scoped>
</style>

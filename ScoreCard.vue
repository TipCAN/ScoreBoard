<template>
  <v-container fluid>
    <div v-if="dev_note">
      <pre>{{dev_note}}</pre>
    </div>
    <br>

    <h1>Latencies</h1>
    <p class="tagline">95th Percentile / 99th Percentile</p>
    <v-data-table
      :headers="headers"
      :items="records"
      :disable-initial-sort="false"
      :pagination.sync="table_page"
      class="elevation-1"
    >
      <template v-slot:items="props">
        <td>{{ props.item.name }}</td>
        <td v-for="key in keys" v-bind:key="key"><span v-html="props.item[key]"></span></td>
      </template>
    </v-data-table>

    <div v-if="chart_latency">
      <v-container fluid>
        <v-layout row>
          <v-flex>
            <v-card>
              <v-flex pl-3 pt-3>
                <div>N.B. any value greater than 1000ms is omitted.</div>
                <div>if you're browsing this on mobile device, feel free to turn 'OFF' the chart below.</div>
                <v-radio-group
                  v-model="chart_latency_source"
                  row
                  v-on:change="switchChartLatencySource"
                >
                  <v-radio label="Off" value="off"></v-radio>
                  <v-radio label="Average" value="avg"></v-radio>
                  <v-radio label="Max" value="max"></v-radio>
                  <v-radio label="95th Percentile" value="p95"></v-radio>
                  <v-radio label="99th Percentile" value="p99"></v-radio>
                  <v-radio label="Success Rate" value="success_rate"></v-radio>
                </v-radio-group>
              </v-flex>
              <v-chart
                :options="chart_latency"
                @click="chartLatencyClick"
                class="echart_latency"
                v-if="chart_latency_source != 'off'"
              />
            </v-card>
          </v-flex>
        </v-layout>
      </v-container>
    </div>

    <br>
    <h1>Experiment Details</h1>

    <v-form>
      <v-container fluid>
        <v-layout>
          <v-flex xs12 md3>
            <v-select
              v-model="select_tag"
              :items="tag_list"
              :rules="[v => !!v || 'Batch size is required']"
              label="Batch Size"
              required
            ></v-select>
          </v-flex>
          <v-flex xs12 md3>
            <v-select
              v-model="select_rps"
              :items="rps_list"
              :rules="[v => !!v || 'QPS is required']"
              label="QPS"
              required
            ></v-select>
          </v-flex>
          <v-flex xs12 md3>
            <v-btn @click="loadDetail">load</v-btn>
          </v-flex>
        </v-layout>
        <p style="color: red;" v-if="form_warning.length > 0">{{form_warning}}</p>
      </v-container>
    </v-form>

    <div v-if="view_record && form_warning.length == 0">
      <v-container fluid>
        <v-layout row>
          <v-flex xs12 md4>
            <v-card>
              <v-toolbar color="blue" dark>
                <v-toolbar-title>Experiment - {{view_record.tag}},qps={{view_record.req_per_sec}}</v-toolbar-title>
              </v-toolbar>

              <v-list subheader two-line>
                <v-subheader>Result</v-subheader>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>Success Rate</v-list-tile-title>
                    <v-list-tile-sub-title>{{(view_record.success_rate*100).toFixed(2)}}%</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>Average Latency</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.latency_avg_ms}}ms</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>Max Latency</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.latency_max_ms}}ms</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>95th Percentile Latency</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.latency_p95_ms}}ms</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>99th Percentile Latency</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.latency_p99_ms}}ms</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>
              </v-list>

              <v-divider></v-divider>

              <v-list subheader two-line>
                <v-subheader>Basic</v-subheader>
                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>Experiment Start Time</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.start_time}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar>
                  <v-list-tile-content>
                    <v-list-tile-title>Experiment End Time</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.end_time}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.total_request">
                  <v-list-tile-content>
                    <v-list-tile-title>Total Requests</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.total_request}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>
              </v-list>

              <v-divider></v-divider>

              <v-list two-line subheader>
                <v-subheader>Configuration</v-subheader>

                <v-list-tile avatar v-if="view_record.api_url">
                  <v-list-tile-content>
                    <v-list-tile-title>API Endpoint URL</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.api_url}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.worker_count">
                  <v-list-tile-content>
                    <v-list-tile-title>Number of Workers</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.worker_count}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.warm_up_sec">
                  <v-list-tile-content>
                    <v-list-tile-title>Warm-up Duration in Seconds</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.warm_up_sec}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.duration_sec">
                  <v-list-tile-content>
                    <v-list-tile-title>Experiment Duration in Seconds</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.duration_sec}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.image_path">
                  <v-list-tile-content>
                    <v-list-tile-title>Image File Path</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.image_path}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.image_size">
                  <v-list-tile-content>
                    <v-list-tile-title>Image File Size</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.image_size}} bytes</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>

                <v-list-tile avatar v-if="view_record.bounding !== undefined">
                  <v-list-tile-content>
                    <v-list-tile-title>With Bounding</v-list-tile-title>
                    <v-list-tile-sub-title>{{view_record.bounding ? "Yes" : "No"}}</v-list-tile-sub-title>
                  </v-list-tile-content>
                </v-list-tile>
              </v-list>
            </v-card>
          </v-flex>

          <v-flex xs12 md8 pl-4>
            <v-card>
              <v-toolbar color="green" dark>
                <v-toolbar-title>Experiment Request Latencies - {{view_record.tag}},qps={{view_record.req_per_sec}}</v-toolbar-title>
              </v-toolbar>
              <v-card-title primary-title>
                <div v-if="chart_experiment">
                  <v-chart :options="chart_experiment"/>
                </div>
              </v-card-title>
            </v-card>

            <br>

            <v-card>
              <v-toolbar color="pink" dark>
                <v-toolbar-title>Warm Up Request Latencies - {{view_record.tag}},qps={{view_record.req_per_sec}}</v-toolbar-title>
              </v-toolbar>
              <v-card-title primary-title>
                <div v-if="chart_warmup">
                  <v-chart :options="chart_warmup"/>
                </div>
              </v-card-title>
            </v-card>
          </v-flex>
        </v-layout>
      </v-container>
    </div>
  </v-container>
</template>

<script>
import json from "../assets/score.json";

export default {
  created() {
    this.switchChartLatencySource(this.chart_latency_source);
  },
  methods: {
    loadDetail() {
      var tag = this.select_tag;
      var rps = this.select_rps;
      if (!tag || !rps) {
        this.form_warning = "batch_size and qps should be selected.";
      } else {
        var record = json.data[tag + "_" + rps];
        if (!record) {
          this.form_warning = `got no experiment record for ${tag}, qps=${rps}.`;
        } else {
          this.form_warning = "";
          this.view_record = record;
          this.updateWarmupChartOptions(record.warmup_sequence || []);
          this.updateExperimentChartOptions(
            record.start_time,
            record.end_time,
            record.latency_sequence || []
          );
        }
      }
    },
    chartLatencyClick(event, instance, echarts) {
      var pos = event.data.value;
      var rps = this.rps_list[pos[0]];
      var tag = this.tag_list[pos[1]];
      this.select_rps = rps;
      this.select_tag = tag;
      this.loadDetail();
    },
    switchChartLatencySource(new_key) {
      if (new_key == "off") {
        return;
      }

      var field_key = "success_rate";
      if (
        new_key == "p95" ||
        new_key == "p99" ||
        new_key == "max" ||
        new_key == "avg"
      ) {
        field_key = `latency_${new_key}_ms`;
      }

      this.updateLatencyChartOptions(field_key);
    },
    updateLatencyChartOptions(field_key) {
      var data = [];
      var self = this;
      Object.values(this.raw_data).forEach(function(record) {
        var x = self.rps_list.indexOf(record["req_per_sec"]);
        var y = self.tag_list.indexOf(record["tag"]);
        var z = record[field_key];
        if (z <= 1000 && record.success_rate == 1) {
          data.push([x, y, z]);
        }
      });

      this.chart_latency = {
        visualMap: {
          max: 500,
          pieces: [
            { gt: 0, lte: 100 },
            { gt: 100, lte: 300 },
            { gt: 300, lte: 500 },
            { gt: 500 }
          ],
          inRange: {
            color: [
              "#B7E9F7",
              "#88d1f1",
              "#3399ff",
              // "#027dff",
              "#0039a9"
            ]
          }
        },
        xAxis3D: {
          type: "category",
          data: this.rps_list,
          name: "QPS",
          axisLine: {
            interval: 0,
          },
        },
        yAxis3D: {
          type: "category",
          data: this.tag_list,
          name: "Batch Size",
          axisLine: {
            interval: 0,
          },
        },
        zAxis3D: {
          type: "value",
          name: "Latency"
        },
        grid3D: {
          boxWidth: 200,
          boxDepth: 100,
          viewControl: {
            distance: 220,
            alpha: 32,
            beta: 14
          },
          light: {
            main: {
              intensity: 1.2
            },
            ambient: {
              intensity: 0.4
            }
          }
        },
        tooltip: {
          trigger: "item",
          hideDelay: 30000,
          formatter: function(params, ticket, callback) {
            var pos = params.data.value;
            var rps = self.rps_list[pos[0]];
            var tag = self.tag_list[pos[1]];
            var record = self.raw_data[tag + "_" + rps];
            var p95 = record.latency_p95_ms;
            var p99 = record.latency_p99_ms;
            var vmax = record.latency_max_ms;
            var vavg = record.latency_avg_ms;
            var rate = (record.success_rate * 100).toFixed(2);
            return `BS: ${tag}<br/>QPS: ${rps}<br/>OK: ${rate}%<br/>Avg: ${vavg}ms<br/>Max: ${vmax}ms<br/>P95: ${p95}ms<br/>P99: ${p99}ms`;
          }
        },
        series: [
          {
            type: "bar3D",
            data: data.map(function(item) {
              return {
                value: [item[0], item[1], item[2]]
              };
            }),
            name: "Latency",
            shading: "color",
            label: {
              show: true,
              position: 'top',
              textStyle: {
                fontSize: 12,
                borderWidth: 1
              }
            },
            itemStyle: {
              opacity: 0.7
            },
            emphasis: {
              label: {
                textStyle: {
                  fontSize: 20,
                  color: "red"
                }
              },
              itemStyle: {
                color: "red"
              }
            }
          }
        ]
      };
    },
    updateWarmupChartOptions(sequence) {
      var count = sequence.length;
      var numberIndexes = [];
      for (var i = 1; i <= count; i++) {
        numberIndexes.push(i);
      }
      this.chart_warmup = {
        xAxis: {
          type: "category",
          data: numberIndexes,
          name: "Requests"
        },
        yAxis: {
          type: "value",
          name: "Latency (ms)"
        },
        tooltip: {
          trigger: "axis",
          formatter: function(params, ticket, callback) {
            var late = params[0].data;
            var index = params[0].dataIndex + 1;
            return `#${index}: ${late}ms`;
          }
        },
        series: [
          {
            data: sequence,
            type: "line",
            smooth: false
          }
        ]
      };
    },
    updateExperimentChartOptions(start_time, end_time, sequence) {
      var count = sequence.length;
      if (count <= 0) {
        return;
      }

      var startTime = new Date(start_time);
      var startTick = startTime.getTime();
      var endTime = new Date(end_time);
      var endTick = endTime.getTime();
      var tickInterval = (endTick - startTick) / count;

      var timeSequenue = [];
      var timeIndexes = [];
      for (var i = 1; i <= count; i++) {
        var indexTime = new Date(startTick + (i - 1) * tickInterval);
        timeSequenue.push([indexTime, sequence[i]]);
      }

      this.chart_experiment = {
        xAxis: {
          type: "time",
          name: "Time"
        },
        yAxis: {
          type: "value",
          name: "Latency (ms)"
        },
        tooltip: {
          trigger: "axis",
          formatter: function(params, ticket, callback) {
            var index = params[0].dataIndex + 1;
            var time = params[0].data[0].toTimeString().split(" ")[0];
            var latency = params[0].data[1];
            return `#${index}<br/>Time: ${time}<br/>Latency: ${latency}ms`;
          }
        },
        series: [
          {
            data: timeSequenue,
            type: "line",
            symbol: "none",
            smooth: false,
            itemStyle: {
              normal: {
                lineStyle: {
                  color: "green",
                  width: 0.5
                }
              }
            }
          }
        ]
      };
    }
  },
  data() {
    var item_keys = [];
    var headers = [
      {
        text: "Batch Size",
        value: "name",
        sortable: false
      }
    ];

    json.rps_list.forEach(function(rps) {
      var title = "qps=" + rps;
      var key = "s" + rps;
      headers.push({
        text: title,
        value: key,
        sortable: false
      });
      item_keys.push(key);
    });

    var rows = [];
    json.tag_list.forEach(function(tag) {
      var row = {
        name: tag
      };
      json.rps_list.forEach(function(rps) {
        var result = "-";
        var record = json.data[tag + "_" + rps];
        if (record) {
          result = `${record.latency_p95_ms}ms/${record.latency_p99_ms}ms`;
          if (record.success_rate < 1) {
            result = "<del>" + result + "</del>";
          }
        }
        row["s" + rps] = result;
      });
      rows.push(row);
    });

    var tablePagination = {
      rowsPerPage: -1,
    };

    return {
      headers: headers,
      records: rows,
      keys: item_keys,
      table_page: tablePagination,
      dev_note: json.note,
      tag_list: json.tag_list,
      rps_list: json.rps_list,
      raw_data: json.data,
      select_rps: null,
      select_tag: null,
      form_warning: "",
      view_record: null,
      chart_warmup: null,
      chart_experiment: null,
      chart_latency: null,
      chart_latency_source: "p99"
    };
  }
};
</script>

<style>
.echart_latency {
  width: 1000px;
  height: 700px;
}
</style>

<template>
    <div>
        
        <div class="period-options">
            <button
                type="button"
                class="btn btn-light dropdown-toggle btn-period-toggle"
                data-bs-toggle="dropdown"
                aria-expanded="false"
            >
                <!-- Shows current selected period -->
                {{ chartPeriodOptions[chartPeriodHrs] }}&nbsp;
            </button>
            <ul class="dropdown-menu dropdown-menu-end">
                <!-- List all period options -->
                <li v-for="(item, key) in chartPeriodOptions" :key="key">
                    <!-- Highlight selected, change period on click -->
                    <a
                        class="dropdown-item"
                        :class="{ active: chartPeriodHrs == key }"
                        href="#"
                        @click="chartPeriodHrs = key"
                        >{{ item }}</a
                    >
                </li>
            </ul>
        </div>
        <!-- Chart area, blurred when loading -->
        <div class="chart-wrapper" :class="{ loading: loading }">
            <!-- Chart.js line chart -->
            <Line :data="chartData" :options="chartOptions" />
        </div>
    </div>
</template>

<script lang="js">
// Import Chart.js modules for drawing charts
import { BarController, BarElement, Chart, Filler, LinearScale, LineController, LineElement, PointElement, TimeScale, Tooltip } from "chart.js";
// Adapter for time axis
import "chartjs-adapter-dayjs-4";
// Import dayjs for date/time handling
import dayjs from "dayjs";
// Import vue-chartjs Line component
import { Line } from "vue-chartjs";
// Toast notifications
import { useToast } from "vue-toastification";
// Status constants and logger
import { DOWN, PENDING, MAINTENANCE, log } from "../util.ts";

// Create toast instance
const toast = useToast();

// Register Chart.js controllers and plugins
Chart.register(LineController, BarController, LineElement, PointElement, TimeScale, BarElement, LinearScale, Tooltip, Filler);

export default {
    components: { Line }, // Register Line chart component
    props: {
        /** ID of monitor */
        monitorId: {
            type: Number,
            required: true, // Must be provided
        },
    },
    data() {
        return {
            loading: false, // True when fetching data

            // Selected chart period (hours)
            chartPeriodHrs: 0,

            // Options for chart period dropdown
            chartPeriodOptions: {
                0: this.$t("recent"),
                3: "3h",
                6: "6h",
                24: "24h",
                168: "1w",
            },

            // Heartbeat data for chart, null means use global list
            heartbeatList: null
        };
    },
    computed: {
        // Chart.js options for rendering the chart
        chartOptions() {
            return {
                responsive: true, // Chart resizes with window
                maintainAspectRatio: false, // Don't keep aspect ratio
                onResize: (chart) => {
                    // Set chart height based on screen size
                    chart.canvas.parentNode.style.position = "relative";
                    if (screen.width < 576) {
                        chart.canvas.parentNode.style.height = "275px";
                    } else if (screen.width < 768) {
                        chart.canvas.parentNode.style.height = "320px";
                    } else if (screen.width < 992) {
                        chart.canvas.parentNode.style.height = "300px";
                    } else {
                        chart.canvas.parentNode.style.height = "250px";
                    }
                },
                layout: {
                    padding: {
                        left: 10,
                        right: 30,
                        top: 30,
                        bottom: 10,
                    },
                },
                elements: {
                    point: {
                        // Hide points unless hovered
                        radius: 0,
                        hitRadius: 100,
                    },
                },
                scales: {
                    x: {
                        type: "time", // Time-based x-axis
                        time: {
                            minUnit: "minute",
                            round: "second",
                            tooltipFormat: "YYYY-MM-DD HH:mm:ss",
                            displayFormats: {
                                minute: "HH:mm",
                                hour: "MM-DD HH:mm",
                            }
                        },
                        ticks: {
                            sampleSize: 3,
                            maxRotation: 0,
                            autoSkipPadding: 30,
                            padding: 3,
                        },
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                            offset: false,
                        },
                    },
                    y: {
                        title: {
                            display: true,
                            text: this.$t("respTime"),
                        },
                        offset: false,
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                        },
                    },
                    y1: {
                        display: false,
                        position: "right",
                        grid: {
                            drawOnChartArea: false,
                        },
                        min: 0,
                        max: 1,
                        offset: false,
                    },
                },
                bounds: "ticks",
                plugins: {
                    tooltip: {
                        mode: "nearest",
                        intersect: false,
                        padding: 10,
                        backgroundColor: this.$root.theme === "light" ? "rgba(212,232,222,1.0)" : "rgba(32,42,38,1.0)",
                        bodyColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        titleColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        filter: function (tooltipItem) {
                            // Only show tooltip for line chart
                            return tooltipItem.datasetIndex === 0;
                        },
                        callbacks: {
                            label: (context) => {
                                // Show ping in ms
                                return ` ${new Intl.NumberFormat().format(context.parsed.y)} ms`;
                            },
                        }
                    },
                    legend: {
                        display: false, // Hide legend
                    },
                },
            };
        },
        // Prepare chart data from heartbeatList
        chartData() {
            let pingData = [];  // Line chart data (ping times)
            let downData = [];  // Bar chart data (status)
            let colorData = []; // Bar chart colors

            // Use local heartbeatList or global one
            let heartbeatList = this.heartbeatList ||
             (this.monitorId in this.$root.heartbeatList && this.$root.heartbeatList[this.monitorId]) ||
             [];

            // Filter and map heartbeats for chart
            heartbeatList
                .filter(
                    // Only keep heartbeats within selected period
                    (beat) => dayjs.utc(beat.time).tz(this.$root.timezone).isAfter(
                        dayjs().subtract(Math.max(this.chartPeriodHrs, 6), "hours")
                    )
                )
                .map((beat) => {
                    // Format time for chart
                    const x = this.$root.datetime(beat.time);
                    // Add ping value
                    pingData.push({
                        x,
                        y: beat.ping,
                    });
                    // Add status value (down/maintenance/pending = 1, up = 0)
                    downData.push({
                        x,
                        y: (beat.status === DOWN || beat.status === MAINTENANCE || beat.status === PENDING) ? 1 : 0,
                    });
                    // Set bar color based on status
                    colorData.push((beat.status === MAINTENANCE) ? "rgba(23,71,245,0.41)" : ((beat.status === PENDING) ? "rgba(245,182,23,0.41)" : "#DC354568"));
                });

            // Return datasets for Chart.js
            return {
                datasets: [
                    {
                        // Line chart (ping)
                        data: pingData,
                        fill: "origin",
                        tension: 0.2,
                        borderColor: "#5CDD8B",
                        backgroundColor: "#5CDD8B38",
                        yAxisID: "y",
                        label: "ping",
                    },
                    {
                        // Bar chart (status)
                        type: "bar",
                        data: downData,
                        borderColor: "#00000000",
                        backgroundColor: colorData,
                        yAxisID: "y1",
                        barThickness: "flex",
                        barPercentage: 1,
                        categoryPercentage: 1,
                        inflateAmount: 0.05,
                        label: "status",
                    },
                ],
            };
        },
    },
    watch: {
  chartPeriodHrs: {
    handler(newPeriod) {
      if (newPeriod == "0") {
        this.heartbeatList = null;
        this.$root.storage().removeItem("chart-period-global");
      } else {
        this.loading = true;
        this.$root.getMonitorBeats(this.monitorId, newPeriod, (res) => {
          if (!res.ok) {
            toast.error(res.msg);
          } else {
            this.heartbeatList = res.data;
            this.$root.storage()["chart-period-global"] = newPeriod;
          }
          this.loading = false;
        });
      }
    },
  }
},

    created() {
        // Watch for changes in global heartbeatList for this monitor
        this.$watch(() => this.$root.heartbeatList[this.monitorId],
            (heartbeatList) => {
                log.debug("ping_chart", `this.chartPeriodHrs type ${typeof this.chartPeriodHrs}, value: ${this.chartPeriodHrs}`);
                // If not "recent", append new heartbeat to local list
                if (this.chartPeriodHrs != "0") {
                    const newBeat = heartbeatList.at(-1);
                    if (newBeat && dayjs.utc(newBeat.time) > dayjs.utc(this.heartbeatList.at(-1)?.time)) {
                        this.heartbeatList.push(heartbeatList.at(-1));
                    }
                }
            },
            { deep: true }
        );

        // On component creation, load chart period from storage if available
        let period = this.$root.storage()["chart-period-global"];
        this.chartPeriodHrs = period != null ? period : "0";
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

// Style for select dropdown
.form-select {
    width: unset;
    display: inline-flex;
}

// Style for period dropdown
.period-options {
    padding: 0.1em 1em;
    margin-bottom: -1.2em;
    float: right;
    position: relative;
    z-index: 10;

    .dropdown-menu {
        padding: 0;
        min-width: 50px;
        font-size: 0.9em;

        .dark & {
            background: $dark-bg;
        }

        .dropdown-item {
            border-radius: 0.3rem;
            padding: 2px 16px 4px;

            .dark & {
                background: $dark-bg;
            }

            .dark &:hover {
                background: $dark-font-color;
                color: $dark-font-color2;
            }
        }

        .dark & .dropdown-item.active {
            background: $primary;
            color: $dark-font-color2;
        }
    }

    .btn-period-toggle {
        padding: 2px 15px;
        background: transparent;
        border: 0;
        color: $link-color;
        opacity: 0.7;
        font-size: 0.9em;

        &::after {
            vertical-align: 0.155em;
        }

        .dark & {
            color: $dark-font-color;
        }
    }
}

// Chart area style
.chart-wrapper {
    margin-bottom: 0.5em;

    &.loading {
        filter: blur(10px); // Blur chart when loading
    }
}
</style>

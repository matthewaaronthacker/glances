<template>
    <!-- prettier-ignore -->
    <section id="processlist-plugin" class="plugin">
        <div class="table">
            <div class="table-row">
                <div
                    class="table-cell"
                    :class="['sortable', sorter.column === 'cpu_percent' && 'sort']"
                    @click="$emit('update:sorter', 'cpu_percent')"
                >
                    CPU%
                </div>
                <div
                    class="table-cell"
                    :class="['sortable', sorter.column === 'memory_percent' && 'sort']"
                    @click="$emit('update:sorter', 'memory_percent')"
                >
                    MEM%
                </div>
                <div class="table-cell hidden-xs hidden-sm">VIRT</div>
                <div class="table-cell hidden-xs hidden-sm">RES</div>
                <div class="table-cell">PID</div>
                <div
                    class="table-cell text-left"
                    :class="['sortable', sorter.column === 'username' && 'sort']"
                    @click="$emit('update:sorter', 'username')"
                >
                    USER
                </div>
                <div
                    class="table-cell hidden-xs hidden-sm"
                    :class="['sortable', sorter.column === 'timemillis' && 'sort']"
                    @click="$emit('update:sorter', 'timemillis')"
                >
                    TIME+
                </div>
                <div
                    class="table-cell text-left hidden-xs hidden-sm"
                    :class="['sortable', sorter.column === 'num_threads' && 'sort']"
                    @click="$emit('update:sorter', 'num_threads')"
                >
                    THR
                </div>
                <div class="table-cell">NI</div>
                <div class="table-cell">S</div>
                <div
                    v-show="ioReadWritePresent"
                    class="table-cell hidden-xs hidden-sm"
                    :class="['sortable', sorter.column === 'io_read' && 'sort']"
                    @click="$emit('update:sorter', 'io_read')"
                >
                    IOR/s
                </div>
                <div
                    v-show="ioReadWritePresent"
                    class="table-cell text-left hidden-xs hidden-sm"
                    :class="['sortable', sorter.column === 'io_write' && 'sort']"
                    @click="$emit('update:sorter', 'io_write')"
                >
                    IOW/s
                </div>
                <div
                    class="table-cell text-left"
                    :class="['sortable', sorter.column === 'name' && 'sort']"
                    @click="$emit('update:sorter', 'name')"
                >
                    Command
                </div>
            </div>
            <div
                class="table-row"
                v-for="(process, processId) in processes"
                :key="processId"
            >
                <div class="table-cell" :class="getCpuPercentAlert(process)">
                    {{ process.cpu_percent == -1 ? '?' : $filters.number(process.cpu_percent, 1) }}
                </div>
                <div class="table-cell" :class="getMemoryPercentAlert(process)">
                    {{ process.memory_percent == -1 ? '?' : $filters.number(process.memory_percent, 1) }}
                </div>
                <div class="table-cell hidden-xs hidden-sm">
                    {{ $filters.bytes(process.memvirt) }}
                </div>
                <div class="table-cell hidden-xs hidden-sm">
                    {{ $filters.bytes(process.memres) }}
                </div>
                <div class="table-cell">
                    {{ process.pid }}
                </div>
                <div class="table-cell text-left">
                    {{ process.username }}
                </div>
                <div class="table-cell hidden-xs hidden-sm" v-if="process.timeplus != '?'">
                    <span v-show="process.timeplus.hours > 0" class="highlight">{{ process.timeplus.hours }}h</span>
                    {{ $filters.leftPad(process.timeplus.minutes, 2, '0') }}:{{ $filters.leftPad(process.timeplus.seconds, 2, '0') }}
                    <span v-show="process.timeplus.hours <= 0">.{{ $filters.leftPad(process.timeplus.milliseconds, 2, '0') }}</span>
                </div>
                <div class="table-cell hidden-xs hidden-sm" v-if="process.timeplus == '?'">?</div>
                <div class="table-cell text-left hidden-xs hidden-sm">
                    {{ process.num_threads == -1 ? '?' : process.num_threads }}
                </div>
                <div class="table-cell" :class="{ nice: process.isNice }">
                    {{ $filters.exclamation(process.nice) }}
                </div>
                <div class="table-cell" :class="{ status: process.status == 'R' }">
                    {{ process.status }}
                </div>
                <div class="table-cell hidden-xs hidden-sm" v-show="ioReadWritePresent">
                    {{ $filters.bytes(process.io_read) }}
                </div>
                <div class="table-cell text-left hidden-xs hidden-sm" v-show="ioReadWritePresent">
                    {{ $filters.bytes(process.io_write) }}
                </div>
                <div class="table-cell text-left" v-show="args.process_short_name">
                    {{ process.name }}
                </div>
                <div class="table-cell text-left" v-show="!args.process_short_name">
                    {{ process.cmdline }}
                </div>
            </div>
        </div>
    </section>
</template>

<script>
import { orderBy, last } from 'lodash';
import { timemillis, timedelta } from '../filters.js';
import { GlancesHelper } from '../services.js';
import { store } from '../store.js';

export default {
    props: {
        data: {
            type: Object
        },
        sorter: {
            type: Object
        }
    },
    data() {
        return {
            store
        };
    },
    computed: {
        args() {
            return this.store.args || {};
        },
        config() {
            return this.store.config || {};
        },
        stats() {
            return this.data.stats['processlist'];
        },
        processes() {
            const { sorter } = this;
            const isWindows = this.data.stats['isWindows'];
            const processes = (this.stats || []).map((process) => {
                process.memvirt = '?';
                process.memres = '?';
                if (process.memory_info) {
                    process.memvirt = process.memory_info[1];
                    process.memres = process.memory_info[0];
                }

                process.timeplus = '?';
                process.timemillis = '?';
                if (process.cpu_times) {
                    process.timeplus = timedelta(process.cpu_times);
                    process.timemillis = timemillis(process.cpu_times);
                }

                if (process.num_threads === null) {
                    process.num_threads = -1;
                }

                if (process.cpu_percent === null) {
                    process.cpu_percent = -1;
                }

                if (process.memory_percent === null) {
                    process.memory_percent = -1;
                }

                process.io_read = null;
                process.io_write = null;

                if (process.io_counters) {
                    process.io_read =
                        (process.io_counters[0] - process.io_counters[2]) /
                        process.time_since_update;
                    process.io_write =
                        (process.io_counters[1] - process.io_counters[3]) /
                        process.time_since_update;
                }

                process.isNice =
                    process.nice !== undefined &&
                    ((isWindows && process.nice != 32) || (!isWindows && process.nice != 0));

                if (Array.isArray(process.cmdline)) {
                    process.cmdline = process.cmdline.join(' ');
                }

                if (process.cmdline === null) {
                    process.cmdline = process.name;
                }

                if (isWindows && process.username !== null) {
                    process.username = last(process.username.split('\\'));
                }

                return process;
            });

            return orderBy(
                processes,
                [sorter.column],
                [sorter.isReverseColumn(sorter.column) ? 'desc' : 'asc']
            ).slice(0, this.limit);
        },
        ioReadWritePresent() {
            return (this.stats || []).some(({ io_counters }) => io_counters);
        },
        limit() {
            return this.config.outputs !== undefined
                ? this.config.outputs.max_processes_display
                : undefined;
        }
    },
    methods: {
        getCpuPercentAlert(process) {
            return GlancesHelper.getAlert('processlist', 'processlist_cpu_', process.cpu_percent);
        },
        getMemoryPercentAlert(process) {
            return GlancesHelper.getAlert('processlist', 'processlist_mem_', process.cpu_percent);
        }
    }
};
</script>
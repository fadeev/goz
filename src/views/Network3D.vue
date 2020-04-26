<template>
  <div id="chart"></div>
</template>

<style scoped>
#chart {
  width: 100vw;
  height: calc(100vh - 3rem);
}
</style>

<script>
import axios from "axios";
import ForceGraph3D from "3d-force-graph";
import { v4 as uuidv4 } from "uuid";
import { find, groupBy } from "lodash";
import io from "socket.io-client";

const API = process.env.VUE_APP_API_URL;

const stringToRGB = string => {
  const hashCode = str => {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = str.charCodeAt(i) + ((hash << 5) - hash);
    }
    return hash;
  };
  const i = hashCode(string);
  let c = (i & 0x00ffffff).toString(16).toUpperCase();
  return "00000".substring(0, 6 - c.length) + c;
};

export default {
  data: function() {
    return {
      txs: [],
      socket: null,
      chart: null,
      relations: {},
      blockchains: []
    };
  },
  watch: {
    txsSendPacket() {
      if (this.chart) {
        this.chart.setOption(this.chartOptions);
      }
    }
  },
  computed: {
    chartData() {
      return {
        nodes: [...this.addressNodes, ...this.blockchainNodes],
        links: [...this.addressLinks, ...this.blockchainLinks]
        // links: [...this.addressLinks]
      };
    },
    txsSendPacket() {
      return this.txs.filter(tx => {
        return find(tx.events, { type: "send_packet" });
      });
    },
    addressNodes() {
      let nodes = [];
      this.txsSendPacket.forEach(tx => {
        const send_packet = find(tx.events, { type: "send_packet" }).attributes;
        const t = JSON.parse(find(send_packet, { key: "packet_data" }).val)
          .value;
        nodes.push(t.receiver);
        nodes.push(t.sender);
      });
      const unique = [...new Set(nodes)];
      return unique.map(addr => {
        return {
          id: addr,
          val: addr,
          name: addr,
          type: "address",
          color: `#${stringToRGB(this.relationsAll[addr] || "unknown")}`
        };
      });
    },
    addressLinks() {
      return this.txsSendPacket.map(t => {
        const send_packet = find(t.events, { type: "send_packet" }).attributes;
        const tx = JSON.parse(find(send_packet, { key: "packet_data" }).val)
          .value;
        return {
          source: tx.sender,
          target: tx.receiver,
          type: "address"
          // color: `#${stringToRGB(tx.sender)}`
        };
      });
    },
    relationsAll() {
      let data = {};
      this.txs.forEach(tx => {
        Object.keys(tx.events).forEach(i => {
          const ev = tx.events[i];
          if (ev.type === "recv_packet") {
            const packet_data = find(ev.attributes, { key: "packet_data" });
            const addr = JSON.parse(packet_data.val).value.receiver;
            data[addr] = tx.domain;
          }
          if (ev.type === "send_packet") {
            const packet_data = find(ev.attributes, { key: "packet_data" });
            const addr = JSON.parse(packet_data.val).value.sender;
            data[addr] = tx.domain;
          }
        });
      });
      return { ...data, ...this.relations };
    },
    blockchainLinks() {
      let links = [];
      Object.keys(this.relationsAll).map(addr => {
        links.push({
          source: this.relationsAll[addr],
          target: addr,
          type: "blockchain"
        });
      });
      links = links.filter(link => {
        // console.log("link.target", link.target);
        return find(this.addressNodes, { id: link.target });
      });
      return links;
    },
    blockchainNodes() {
      let nodes = [];
      this.blockchains.map(c => {
        nodes.push({
          id: c,
          val: c,
          name: c,
          type: "blockchain",
          color: `#${stringToRGB(c)}`
        });
      });
      return nodes;
    }
  },
  async mounted() {
    this.socket = io(`${API}`);
    this.socket.on("tx", tx => {
      console.log(tx);
      this.txs.push(tx);
    });
    this.txs = (await axios.get(`${API}/txs/ibc`)).data;
    this.relations = (await axios.get(`${API}/relations`)).data;
    this.blockchains = (await axios.get(`${API}/blockchains`)).data;

    //console.log("blockchainLinks", this.blockchainLinks);

    let graph = ForceGraph3D()
      .forceEngine("ngraph")
      .nodeAutoColorBy("color")
      .nodeLabel(node => {
        if (node.type === "blockchain") {
          return `[zone] ${node.id}`;
        }
        return node.id;
      })
      .nodeResolution(node => {
        if (node.type === "address") {
          return 0.25;
        }
        return 16;
      })
      .nodeVal(node => {
        if (node.type === "address") {
          return 0.25;
        }
        return 16;
      })
      .linkCurvature(link => {
        if (link.type === "address") {
          return 0.25;
        }
        return 0;
      })
      .linkDirectionalArrowLength(link => {
        if (link.type === "address") {
          return 2;
        }
        return 0;
      })
      .linkOpacity(0.5)
      .linkWidth(link => {
        if (link.type === "address") {
          return 0.5;
        }
        return 0.25;
      });

    graph(document.getElementById("chart")).graphData(this.chartData);
  }
};
</script>
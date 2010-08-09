---
layout: article
title: check isp and ccdc registers with json
categories: ccdc isp
updated_at: 08,2010-06
---

Goal
====

Reduce typing and thinking, increase getting'r'done-itude.

Feed in an array of registers to read, get a printout on the screen.

Feed in an array of registers with values, get a printout of success / error.

Script
======
`ccdc_reg.js`

    #!/usr/bin/env node
    var registers = {},
      devreg = require('./devreg').devreg,
      device = require('./ccdc').ccdc,
      settings = require('./settings').settings,
      Futures = require('./futures');
    
    //devreg(device, device.base_addr.omap3530);
    devreg(device, device.base_addr, settings);

`devreg.js`

    #!/usr/bin/env node
    var g_registers = {},
        sys = require('sys'),
        exec = require('child_process').exec,
        device,
        settings,
        base_addr,
        Futures = require('./futures');
        // wget http://github.com/coolaj86/futures/raw/v0.9.0/lib/futures.js
        // or
        // npm install futures

    // TODO: put in the 'Zen' enumerate functions, not magic parameters
    function leadWith(string, len, char) {
      char = ('undefined' !== typeof char) ? char : ' ';
      string = string || '';
      string = '' + string;
      while (string.length < len) {
        string = char + string;
      }
      return string;
    }

    function trailWith(string, len, char) {
      char = ('undefined' !== typeof char) ? char : ' ';
      string = string || '';
      string = '' + string;
      while (string.length < len) {
        string += char;
      }
      return string;
    }

    function print_nibbles(reg_name, value) {
      var bits = device.bits[reg_name],
        bit_name,
        bit,
        shift,
        len,
        bin;

      //sys.print(bits + ' ' + reg_name + ' ' + value);
      sys.print("\n");
      sys.print(leadWith('', 22, ' ') + leadWith('-', 12+8+8, '-') + "\n");
      for (bit_name in bits) {
        if (!bits.hasOwnProperty(bit_name)) return;

        shift = bits[bit_name];

        bit_name = trailWith(bit_name + ' [' + shift.join(':') + '] ', 20);
        len = shift[1] && 1 + shift[1] - shift[0] || 1;

        bin = leadWith(value.toString(2), 32, '0');
        nibble = bin.substr(shift[0], len);

        //sys.print('  ' + bit_name + ' bin' + bin + 'shift:' + shift.join(',') + ' ' + len + ' nib:' + nibble + "\n");

        hex = leadWith(parseInt(nibble, 2).toString(16), 8, '0');
        dec = parseInt(nibble, 2).toString(10);

        sys.print('  ' + bit_name + " 0x" + hex + " == " + dec + "\n");
      }
      sys.print(leadWith('', 22, ' ') + leadWith('-', 12+8+8, '-') + "\n");
      sys.print("\n\n");
    }

    function print_registers() {
      var reg_name,
        value,
        dec, // decimal
        hex,
        bin;

      for (reg_name in g_registers) {
        if (!g_registers.hasOwnProperty(reg_name)) return;

        value = g_registers[reg_name];
        dec = leadWith(value.toString(10), 12, '0');
        hex = leadWith(value.toString(16), 8, '0');
        bin = leadWith(value.toString(2), 32, '0');
        
        key = trailWith(reg_name + ':', 20);
        sys.print(key.toUpperCase() + '   0x' + hex + ' == ' + dec + ' | 0b' + bin + "\n");
        print_nibbles(reg_name, value);
      }
    }

    function write_settings() {
      var bits,
        reg_name,
        value;

      //sys.print( ' ' + JSON.stringify(settings) + "\n");

      for (reg_name in settings) {
        if (!settings.hasOwnProperty(reg_name)) return;
        if (!g_registers.hasOwnProperty(reg_name)) throw new Exception("'" + k + "' isn't a known register");
        value = settings[reg_name];
        
        if ('number' == typeof value) {
          // awesome
        } else if ('string' == typeof value) {
          value = value.toLowerCase();
          if (0 === value.indexOf('0x')) {
            value = parseInt(value.substr(2), 16);
          } else if (0 === value.indexOf('0b')) {
            value = parseInt(value.substr(2), 2);
          } else {
            value = parseInt(value, 10);
          }
        } else if ('object' == typeof value) {
          throw new Exception('hashmaps of bits for writing Not supported yet');
        } else {
          throw new Exception('values in settings should be in the form "0x00000000", "0b00000000", decimal, or key/value pairs of bits');
        }

        devmem(reg_name, '0x' + value.toString(16), 'u32');
      }
    }

    // TODO platform abstract and move to devmem.js
    function devmem(reg_name, hex_value, size) {
      var promise = Futures.promise();
      k, // key
      v, // value
      hexaddr = (base_addr.substr(0, 8) + device.registers[reg_name].substr(2)),
      hex,
      bin,
      dec;

      hex_value = ('undefined' !== typeof hex_value) ? hex_value : '';
      exec('devmem2 ' + hexaddr + ' w ' + hex_value + ' | grep : | cut -d":" -f2', function (error, stdout, stderr) {
      	//sys.print('devmem2 ' + hexaddr + ' w ' + hex_value + ' | grep : | cut -d":" -f2' + "\n");

        if (stderr) {
          sys.print('stderr: ' + stderr);
          throw new Exception('yipes');
        }

        if (error !== null) {
          console.log('exec error: ' + error);
        }

        // TODO handle other formats?
        stdout = stdout.substr(3); // removing leading ' 0x'
        dec = parseInt(stdout, 16);
        g_registers[reg_name] = dec;
        promise.fulfill({reg_name : dec});
      });
      return promise;
    }

    function devreg(_device, _base_addr, _settings) {
      var promise = Futures.promise();
        promises = [],
        join,
        key,
        value,
        hexaddr;

      device = _device;
      base_addr = _base_addr || device.base_addr;
      settings = _settings || {};

      //sys.puts(JSON.stringify(device) + "\n");
      for (key in device.registers) {
        if (!device.registers.hasOwnProperty(key)) return;
        //value = device.registers[key];
        //sys.print(key + ": " + value.substr(2) + "\n");
        promises.push(devmem(key));
      }
      join = Futures.join(promises);
      // TODO validate writes
      join
        .when(write_settings)
        .when(print_settings)
        .when(function () {
          promise.fulfill(g_registers);
        });

      return promise;
    }

    export.devreg = devreg;

`settings.js`

    // TODO allow setting of user values
    // will be checked
    // registers can be set with decimal, hex, or binary value
    exports.settings = {
      "registers" : {
        "pid": 0, // these types are implemented
        "pcr": 0x0,
        "syn_mode": 0b0, 
        "hd_vd_wid": { // Not implemented
          "reserved": 0,
          "hdw": 0x0,
          "reserved": 0b0,
          "vdw": 0
        },
        "pix_lines": 1,
        "horz_info": 1,
        "vert_start": 1,
        "vert_lines": 1,
        "culling": 1,
        "hsize_off": 1,
        "sdofst": 1,
        "sdr_addr": 1,
        "clamp": 1,
        "dcsub": 1,
        "colptn": 1,
        "blkcmp": 1,
        "fpc": 1,
        "fpc_addr": 1,
        "vdint": 1,
        "alaw": 1,
        "rec656if": 1,
        "cfg": 1,
        "fmtcfg": 1,
        "fmt_horz": 1,
        "fmt_vert": 1,
        "fmt_addr_i": 1,
        "prgeven0": 1,
        "prgeven1": 1,
        "prgodd0": 1,
        "prgodd1": 1,
        "vp_out": 1,
        "lsc_config": 1,
        "lsc_initial": 1,
        "lsc_table_base": 1,
        "lsc_table_offset": 1
      }
    }


`ccdc.js`

    // TODO add reset values as to be able 
    // to check for hardware bugs as well
    var ccdc = {
      "base_addr" : "0x480BC600",
      "registers" : {
        "pid": "0x00",
        "pcr": "0x04",
        "syn_mode": "0x08", 
        "hd_vd_wid": "0x0c",
        "pix_lines": "0x10",
        "horz_info": "0x14",
        "vert_start": "0x18",
        "vert_lines": "0x1c",
        "culling": "0x20",
        "hsize_off": "0x24",
        "sdofst": "0x28",
        "sdr_addr": "0x2c",
        "clamp": "0x30",
        "dcsub": "0x34",
        "colptn": "0x38",
        "blkcmp": "0x3c",
        "fpc": "0x40",
        "fpc_addr": "0x44",
        "vdint": "0x48",
        "alaw": "0x4c",
        "rec656if": "0x50",
        "cfg": "0x54",
        "fmtcfg": "0x58",
        "fmt_horz": "0x5c",
        "fmt_vert": "0x60",
        "fmt_addr_i": "0x64",
        "prgeven0": "0x84",
        "prgeven1": "0x88",
        "prgodd0": "0x8c",
        "prgodd1": "0x90",
        "vp_out": "0x94",
        "lsc_config": "0x98",
        "lsc_initial": "0x9c",
        "lsc_table_base": "0xa0",
        "lsc_table_offset": "0xa4"
      },
      "bits": {
        "pid": {
          "reserved": [24,31],
          "tid": [16,31],
          "cid": [8,15],
          "prev": [0,7],
        },
        "pcr": {
          "reserved": [2,31],
          "busy": [1],
          "enable": [0],
        },
        "syn_mode": {
          "reserved": [20,31],
          "sdr2rsz": [19],
          "vp2sdr": [18],
          "wen" : [17],
          "vdhden": [16],
          "fldstat": [15],
          "lpf": [14],
          "inpmod": [12,13],
          "pack8": [11],
          "datsiz": [8,10],
          "fldmode": [7],
          "datapol": [6],
          "exwen": [5],
          "fldpol": [4],
          "hdpol": [3],
          "vdpol": [2],
          "fldout": [1],
          "vdhdout": [0]
        }, 
        "hd_vd_wid": {
          "reserved": [28,31],
          "hdw": [16,27],
          "reserved": [12,15],
          "vdw": [0,11]
        }, 
        "pix_lines": {
          "ppln": [16,31],
          "hlprf": [0,15]
        }, 
        "horz_info": {
          "reserved": [31],
          "sph": [16,30],
          "reserved": [15],
          "nph": [0,14]
        }, 
        "vert_start": {
          "reserved": [31],
          "slv0": [16,30],
          "reserved": [15],
          "slv1": [0,14]
        }, 
        "vert_lines": {
          "reserved": [15,31],
          "nlv": [0,14],
        },
        "culling": {
          "culhevn": [24,31],
          "culhodd": [16,23],
          "reserved": [8,15],
          "culv": [0,7]
        }, 
        "hsize_off": {
          "reserved": [16,31],
          "lnofst": [0,15]
        }, 
        "sdofst": {
          "reserved": [15,31],
          "fiinv": [14],
          "fofst": [12,13],
          "lofst0": [9,11],
          "lofst1": [6,8],
          "lofst2": [3,5],
          "lofst3": [0,2]
        }, 
        "sdr_addr": {
          "addr": [0,31]
        }, 
        "clamp": {
          "clampen": [31],
          "obslen": [28,30],
          "obsln": [25,27],
          "obst": [10,24],
          "reserved": [5,9],
          "obgain": [0,4]
        }, 
        "dcsub": {
          "reserved": [14,31],
          "dcsub": [0,13]
        }, 
        "colptn": {
          "cp3lpc3": [30,31],
          "cp3lpc2": [28,29],
          "cp3lpc1": [26,27],
          "cp3lpc0": [24,25],
          "cp2plc3": [22,23],
          "cp2plc2": [20,21],
          "cp2plc1": [18,19],
          "cp2plc0": [16,17],
          "cp1plc3": [14,15],
          "cp1plc2": [12,13],
          "cp1plc1": [10,11],
          "cp1plc0": [8,9],
          "cp0plc3": [6,7],
          "cp0plc2": [4,5],
          "cp0plc1": [2,3],
          "cp0plc0": [0,1]
        }, 
        "blkcmp": {
          "r_ye": [24,31],
          "gr_cy": [16,23],
          "gb_g": [8,15],
          "b_mg": [0,7]
        }, 
        "fpc": {
          "reserved": [17,31],
          "fperr": [16],
          "gb_g": [15],
          "b_mg": [0,14]
        }, 
        "fpc_addr": {
          "addr": [0,31]
        }, 
        "vdint": {
          "reserved": [31],
          "vdint0": [16,30],
          "reserved": [15],
          "vdint1": [0,14]
        }, 
        "alaw": {
          "reserved": [4,31],
          "ccdtbl": [3],
          "gwdi": [0,2]
        }, 
        "rec656if": {
          "reserved": [2,31],
          "eccfvh": [1],
          "r656on": [0]
        }, 
        "cfg": {
          "reserved": [16,31],
          "vdlc": [15], 
          "reserved": [14],
          "msbinvi": [13],
          "bswd": [12],
          "y8pos": [11],
          "reserved": [9,10],
          "wenlog": [8],
          "fidmd": [6,7],
          "bw656": [5],
          "reserved": [4],
          "reserved": [3],
          "reserved": [2],
          "reserved": [0,1]
        }, 
        "fmtcfg": {
          "reserved": [19,31],
          "vpif_frq": [16,18],
          "vpen": [15],
          "vpin": [12,14],
          "plen_even": [8,11],
          "plen_odd": [4,7],
          "lnum": [2,3],
          "lnalt": [1],
          "fmten": [0]
        }, 
        "fmt_horz": {
          "reserved": [29,31],
          "fmtsph": [16,28],
          "reserved": [13,15],
          "fmtlnh": [0,12]
        }, 
        "fmt_vert": {
          "reserved": [29,31],
          "fmtslv": [16,28],
          "reserved": [13,15],
          "fmtlnv": [0,12]
        }, 
        "fmt_addr_i": {
          "reserved": [26,31],
          "line": [24,25],
          "reserved": [13,23],
          "init": [0,12]
        }, 
        "prgeven0": {
          "even7": [28,31],
          "even6": [24,27],
          "even5": [20,23],
          "even4": [16,19],
          "even3": [12,15],
          "even2": [8,11],
          "even1": [4,7],
          "even0": [0,3]
        }, 
        "prgeven1": {
          "even15": [28,31],
          "even14": [24,27],
          "even13": [20,23],
          "even12": [16,19],
          "even11": [12,15],
          "even10": [8,11],
          "even9": [4,7],
          "even8": [0,3]
        }, 
        "prgodd0": {
          "odd7": [28,31],
          "odd6": [24,27],
          "odd5": [20,23],
          "odd4": [16,19],
          "odd3": [12,15],
          "odd2": [8,11],
          "odd1": [4,7],
          "odd0": [0,3]
        }, 
        "prgodd1": {
          "odd15": [28,31],
          "odd14": [24,27],
          "odd13": [20,23],
          "odd12": [16,19],
          "odd11": [12,15],
          "odd10": [8,11],
          "odd09": [4,7],
          "odd08": [0,3]
        }, 
        "vp_out": {
          "reserved": [31],
          "vert_num": [17,30],
          "horz_num": [4,16],
          "horz_st": [0,3]
        }, 
        "lsc_config": {
          "reserved": [15,31],
          "gain_mode_m": [12,14],
          "reserved": [11],
          "gain_mode_n": [8,10],
          "busy": [7],
          "after_reform": [6],
          "reserved": [4,5],
          "gain_format": [1,3],
          "enable": [0]
        }, 
        "lsc_initial": {
          "reserved": [22,31],
          "y": [16,21],
          "reserved": [6,15],
          "x": [0,5]
        }, 
        "lsc_table_base": {
          "base": [0,31]
        }, 
        "lsc_table_offset": {
          "reserved": [16,31],
          "offset": [0,15]
        }
      }
    }
    exports.ccdc = ccdc;

---
layout: article
title: Check isp and ccdc registers with json
categories: isp-ccdc
updated_at: 2010-08-09
---

Goal
====

Reduce typing and thinking, increase getting'r'done-itude.

Feed in an array of registers to read, get a printout on the screen.

Feed in an array of registers with values, get a printout of success / error.

TODO: best way to deal with multiple "reserved" bits?

Script
======
`ccdc-reg.js`

    #!/usr/bin/env node
    var devreg = require('./devreg').devreg,
      devices = require('./ispccdc'),
      flattenDocs = require('./flattendocs').flattenDocs,
      settings = require('./settings').settings,
      Futures = require('./futures');
    
    //devreg(device, device.base_addr.omap3530);
    Objects.keys(devices).each(function (deviceName, i, arr) {
      device = flattenDocs(devices[deviceName]);
      devreg(device, device.base_addr, settings)
        .when(function (register_values) {
          // do nothing right now
        });
    });


`devreg.js`

    var physical_registers = {},
        sys = require('sys'),
        parseNumber = require('parse_number').parseNumber,
        exec = require('child_process').exec,
        device,
        settings,
        base_addr,
        global_msb = 32 - 1,
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

    // TODO
    // print '0x    0f1 ' instead of '0x000000f1'
    function format_field(value, shift, len, size) {
      var i = 0, string = ''+value;
      string = leadWith(string, shift+len);
      string = trailWith(string, size);
    }

    function print_nibbles(reg_name, value) {
      var bits = device.fields[reg_name],
        field_name,
        bit,
        slice,
        shift,
        len,
        bin;

      //sys.print(bits + ' ' + reg_name + ' ' + value);
      sys.print("\n");
      sys.print(leadWith('', 22, ' ') + leadWith('-', 12+8+8, '-') + "\n");
      for (field_name in bits) {
        if (!bits.hasOwnProperty(field_name)) return;

        len = 0;
        slice = bits[field_name];
        // interpret correctly no matter the order [0:2] or [3:1] or [4]
        if ('undefined' !== typeof slice[1]) {
          if (slice[1] > slice[0]) {
            shift = slice[1];
            len = slice[1] - slice[0];
          } else {
            shift = slice[0];
            len = slice[0] - slice[1];
          }
        } else {
          shift = slice[0];
        }
        len += 1;

        // TODO allow other byte orders?
        // Because most systems are LSB, if we want to
        // read bit 31:29, from a 32-bit register
        //  we must substr char 0:2
        if (true /* == lsb */) {
          shift = global_msb - shift;
        }

        field_name = trailWith(field_name + ' [' + slice.join(':') + '] ', 20);

        bin = leadWith(value.toString(2), 32, '0');
        nibble = bin.substr(shift, len);

        //sys.print('  ' + field_name + ' bin' + bin + 'slice:' + slice.join(',') + ' ' + len + ' nib:' + nibble + "\n");

        hex = leadWith(parseInt(nibble, 2).toString(16), 8, '0');
        dec = parseInt(nibble, 2).toString(10);

        sys.print('  ' + field_name + " 0x" + hex + " == " + dec + "\n");
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

      for (reg_name in physical_registers) {
        if (!physical_registers.hasOwnProperty(reg_name)) return;

        value = physical_registers[reg_name];
        dec = leadWith(value.toString(10), 12, '0');
        hex = leadWith(value.toString(16), 8, '0');
        bin = leadWith(value.toString(2), 32, '0');
        
        key = trailWith(reg_name + ':', 20);
        sys.print(key.toUpperCase() + '   0x' + hex + ' == ' + dec + ' | 0b' + bin + "\n");
        print_nibbles(reg_name, value);
      }
    }

    // slice example: [3:1]
    function getBinaryFieldData(slice, register_size, field_value) {
      var len = 0,
        mask = 1,
        last_bit = register_size - 1,
        shift,
        i;

      field_value = field_value || 0;

      // interpret correctly no matter the order [0:2] or [3:1] or [4]
      if ('undefined' !== typeof slice[1]) {
        // shift should be the lowest number
        if (slice[1] > slice[0]) {
          shift = slice[0];
          len = slice[1] - slice[0];
        } else {
          shift = slice[1];
          len = slice[0] - slice[1];
        }
      } else {
        shift = slice[0];
      }
      // The length must be at least 1
      len += 1;

      // create a bit mask of all 1s
      // Note: mask is already 1 to start with
      for (i = 1; i < len; i += 1) {
        mask <<= i;
        mask &= 1;
      }

      if (field_value > mask) {
        // TODO would this work for negative numbers? Wolud you use a negative number in a register?
        throw new Error('cowardly refusing to set ' + field_name + ' to "' + field_value + '" when the max value is "' + mask + '"');
      }

      // move the mask and the value into the right place
      for (i = 0; i <= shift; i += 1) {
        field_value <<= i;
        mask <<= i;
      }

      if (mask != (mask & field_value)) {
        throw new Error('Logic error "mask != (mask & field_value"');
      }

      return {
        // consider an 8-bit register
        and_mask: ~mask, // 11110011 resets the 2nd and 3rd bit
        or_value: field_value, // 00001100 ready to be ORed with the full register
        shift: shift, // 2 bits in (right to left), the 2nd and 3rd bit
        length: len, // 2 bits long
        substr_start: last_bit - (shift + (len - 1)) // 4 bits in (left to right) 0000 1100
      };
    }

    function andOrBinaryRegisterValue(reg_name, register) {
      var register_value = physical_registers[reg_name] || 0, 
        field_vals, 
        documentation = device.fields[reg_name];

      if ('undefined' === typeof documentation) {
        throw new Error('"' + reg_name + '"' undefined. Please check spelling and/or update the documentation file');
      }

      // The user was only interested it the value as a whole
      if ('number' === typeof register) {
        register_value = register;
        return register_value;
      }
      if ('string' === typeof register) {
        register_value = parseNumber(register);
        return register_value;
      }

      // The user wants to and & or the new value with the old value on a field-by-field basis
      if ('object' === typeof register) {
        Object.keys(register).forEach(function (field_name, i, arr) {
          var field;
          if ('undefined' === typeof documentation[field_name]) {
            throw new Error('"' + reg_name + ':' + field_name + '"' undefined. Please check spelling and/or update the documentation file');
          }
          field = getBinaryFieldData(documentation[field_name], global_msb + 1, register[field_name]);
          register_value &= field.and_mask; // set the field to 0
          register_value |= field.or_value; // set the field to the value
        });
        return register_value;
      }

      throw new Error('value in "settings:' + reg_name + '" should be in the form "0x00000000", "0b00000000", decimal, or key/value pairs of bits');
      //throw new Error('unexpected type ' + typeof register + ' for "' + JSON.stringify(register) + '"');
    }

    function write_settings() {
      var reg_name,
        register_value;

      //sys.print( ' ' + JSON.stringify(settings) + "\n");

      for (reg_name in settings) {
        if (!settings.hasOwnProperty(reg_name)) return;
        if (!physical_registers.hasOwnProperty(reg_name)) throw new Error("'" + reg_name + "' isn't a known register");

        register_value = andOrBinaryRegisterValue(reg_name, settings[reg_name]);

        devmem(reg_name, '0x' + register_value.toString(16), 'u32').when(function () {
          if (register_value !== physical_registers[reg_name]) {
            sys.puts('ERR: "' + reg_name + '" was written as "' + register_value + '" but read as "' + physical_registers[reg_name]  + '"');
          }
        });
      }
    }

    function assert_settings() {
      var reg_name,
        register_value;

      //sys.print( ' ' + JSON.stringify(settings) + "\n");

      for (reg_name in settings) {
        if (!settings.hasOwnProperty(reg_name)) return;
        if (!physical_registers.hasOwnProperty(reg_name)) throw new Error("'" + k + "' isn't a known register");

        register_value = andOrBinaryRegisterValue(reg_name, settings[reg_name]);
        if (register_value !== (register_value & physical_registers[reg_name])) {
          // TODO compare fields
          sys.puts('ERR: "' + reg_name + '" was expected to be "' + register_value + '" but is actualy "' + physical_registers[reg_name]  + '"');
        }
      }
    }

    // TODO platform abstract and move to devmem.js
    function devmem(reg_name, hex_value, size) {
      var promise = Futures.promise(),
        k, // key
        v, // value
        hexaddr = (parseNumber(base_addr) | parseNumber(device.registers[reg_name]));
        hex,
        bin,
        dec;

      hex_value = ('undefined' !== typeof hex_value) ? hex_value : '';
      exec('devmem2 ' + hexaddr + ' w ' + hex_value + ' | grep : | cut -d":" -f2', function (error, stdout, stderr) {
      	//sys.print('devmem2 ' + hexaddr + ' w ' + hex_value + ' | grep : | cut -d":" -f2' + "\n");

        if (stderr) {
          sys.print('stderr: ' + stderr);
          throw new Error("Couldn't access register: " + stderr);
        }

        if (error !== null) {
          console.log('exec error: ' + error);
        }

        // TODO handle other formats?
        stdout = stdout.substr(3); // removing leading ' 0x'
        dec = parseInt(stdout, 16);
        physical_registers[reg_name] = dec;
        promise.fulfill({reg_name : dec});
      });
      return promise;
    }

    function devreg(_device, _base_addr, _settings) {
      var promise = Futures.promise(),
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
        .when(print_registers)
        .when(function () {
          promise.fulfill(physical_registers);
        });

      return promise;
    }

    exports.devreg = devreg;

`flattendocs.js`

    // Work in Progress

`settings.js`

    // TODO allow setting of user values
    // will be checked
    // registers can be set with decimal, hex, or binary value
    var settings = {
      "registers" : {
        "pid": 0, // these types are implemented
        "pcr": 0x0,
        "syn_mode": 0b0, 
        "hd_vd_wid": { // Not implemented
          "reserved-0": 0,
          "hdw": 0x0,
          "reserved-1": 0b0,
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
    exports.settings = settings;


`ispccdc.js`

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
      "fields": {
        "pid": {
          "reserved": [31,24],
          "tid": [23,16],
          "cid": [15,8],
          "prev": [7,0],
        },
        "pcr": {
          "reserved": [31,2],
          "busy": [1],
          "enable": [0],
        },
        "syn_mode": {
          "reserved": [31,20],
          "sdr2rsz": [19],
          "vp2sdr": [18],
          "wen" : [17],
          "vdhden": [16],
          "fldstat": [15],
          "lpf": [14],
          "inpmod": [13,12],
          "pack8": [11],
          "datsiz": [10,8],
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
          "reserved-0": [31,28],
          "hdw": [27,16],
          "reserved-1": [15,12],
          "vdw": [11,0]
        }, 
        "pix_lines": {
          "ppln": [31,16],
          "hlprf": [15,0]
        }, 
        "horz_info": {
          "reserved-0": [31],
          "sph": [30,16],
          "reserved-1": [15],
          "nph": [14,0]
        }, 
        "vert_start": {
          "reserved-0": [31],
          "slv0": [30,16],
          "reserved-1": [15],
          "slv1": [14,0]
        }, 
        "vert_lines": {
          "reserved": [31,15],
          "nlv": [14,0],
        },
        "culling": {
          "culhevn": [31,24],
          "culhodd": [23,16],
          "reserved": [15,8],
          "culv": [7,0]
        }, 
        "hsize_off": {
          "reserved": [31,16],
          "lnofst": [15,0]
        }, 
        "sdofst": {
          "reserved": [31,15],
          "fiinv": [14],
          "fofst": [13,12],
          "lofst0": [11,9],
          "lofst1": [8,6],
          "lofst2": [5,3],
          "lofst3": [2,0]
        }, 
        "sdr_addr": {
          "addr": [31,0]
        }, 
        "clamp": {
          "clampen": [31],
          "obslen": [30,28],
          "obsln": [27,25],
          "obst": [24,10],
          "reserved": [9,5],
          "obgain": [4,0]
        }, 
        "dcsub": {
          "reserved": [31,14],
          "dcsub": [13,0]
        }, 
        "colptn": {
          "cp3lpc3": [31,30],
          "cp3lpc2": [29,28],
          "cp3lpc1": [27,26],
          "cp3lpc0": [25,24],
          "cp2plc3": [23,22],
          "cp2plc2": [21,20],
          "cp2plc1": [19,18],
          "cp2plc0": [17,16],
          "cp1plc3": [15,14],
          "cp1plc2": [13,12],
          "cp1plc1": [11,10],
          "cp1plc0": [9,8],
          "cp0plc3": [7,6],
          "cp0plc2": [5,4],
          "cp0plc1": [3,2],
          "cp0plc0": [1,0]
        }, 
        "blkcmp": {
          "r_ye": [31,24],
          "gr_cy": [23,16],
          "gb_g": [15,8],
          "b_mg": [7,0]
        }, 
        "fpc": {
          "reserved": [31,17],
          "fperr": [16],
          "gb_g": [15],
          "b_mg": [14,0]
        }, 
        "fpc_addr": {
          "addr": [31,0]
        }, 
        "vdint": {
          "reserved-0": [31],
          "vdint0": [30,16],
          "reserved-1": [15],
          "vdint1": [14,0]
        }, 
        "alaw": {
          "reserved": [31,4],
          "ccdtbl": [3],
          "gwdi": [2,0]
        }, 
        "rec656if": {
          "reserved": [31,2],
          "eccfvh": [1],
          "r656on": [0]
        }, 
        "cfg": {
          "reserved-0": [31,16],
          "vdlc": [15], 
          "reserved-1": [14],
          "msbinvi": [13],
          "bswd": [12],
          "y8pos": [11],
          "reserved-2": [10,9],
          "wenlog": [8],
          "fidmd": [7,6],
          "bw656": [5],
          "reserved-3": [4],
          "reserved-4": [3],
          "reserved-5": [2],
          "reserved-6": [1,0]
        }, 
        "fmtcfg": {
          "reserved": [31,19],
          "vpif_frq": [18,16],
          "vpen": [15],
          "vpin": [14,12],
          "plen_even": [11,8],
          "plen_odd": [7,4],
          "lnum": [3,2],
          "lnalt": [1],
          "fmten": [0]
        }, 
        "fmt_horz": {
          "reserved-0": [31,29],
          "fmtsph": [28,16],
          "reserved-1": [15,13],
          "fmtlnh": [12,0]
        }, 
        "fmt_vert": {
          "reserved-0": [31,29],
          "fmtslv": [28,16],
          "reserved-1": [15,13],
          "fmtlnv": [12,0]
        }, 
        "fmt_addr_i": {
          "reserved-0": [31,26],
          "line": [25,24],
          "reserved-1": [23,13],
          "init": [12,0]
        }, 
        "prgeven0": {
          "even7": [31,28],
          "even6": [27,24],
          "even5": [23,20],
          "even4": [19,16],
          "even3": [15,12],
          "even2": [11,8],
          "even1": [7,4],
          "even0": [3,0]
        }, 
        "prgeven1": {
          "even15": [31,28],
          "even14": [27,24],
          "even13": [23,20],
          "even12": [19,16],
          "even11": [15,12],
          "even10": [11,8],
          "even9": [7,4],
          "even8": [3,0]
        }, 
        "prgodd0": {
          "odd7": [31,28],
          "odd6": [27,24],
          "odd5": [23,20],
          "odd4": [19,16],
          "odd3": [15,12],
          "odd2": [11,8],
          "odd1": [7,4],
          "odd0": [3,0]
        }, 
        "prgodd1": {
          "odd15": [31,28],
          "odd14": [27,24],
          "odd13": [23,20],
          "odd12": [19,16],
          "odd11": [15,12],
          "odd10": [11,8],
          "odd09": [7,4],
          "odd08": [3,0]
        }, 
        "vp_out": {
          "reserved": [31],
          "vert_num": [30,17],
          "horz_num": [16,4],
          "horz_st": [3,0]
        }, 
        "lsc_config": {
          "reserved-0": [31,15],
          "gain_mode_m": [14,12],
          "reserved-1": [11],
          "gain_mode_n": [10,8],
          "busy": [7],
          "after_reform": [6],
          "reserved-2": [5,4],
          "gain_format": [3,1],
          "enable": [0]
        }, 
        "lsc_initial": {
          "reserved-0": [31,22],
          "y": [21,16],
          "reserved-1": [15,6],
          "x": [5,0]
        }, 
        "lsc_table_base": {
          "base": [31,0]
        }, 
        "lsc_table_offset": {
          "reserved": [31,16],
          "offset": [15,0]
        }
      }
    };
    var isp = {
    };
    exports.ccdc = ccdc;
    exports.isp = isp;

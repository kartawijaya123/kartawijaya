/************************************************************
 * Kartawijaya Monitoring — FINAL
 * Google Apps Script
 *
 * DATA:
 *   FMS Productivity API -> Conveyor / MP / Throughput /
 *                           Productivity / Operator
 *   FMS SOC API          -> SOC / Transported / SOC Received
 *
 * SHIFT:
 *   Shift 1 = 00:00 - 08:00
 *   Shift 2 = 08:00 - 16:00
 *   Shift 3 = 16:00 - 24:00
 *
 * AUTH:
 *   FMS_COOKIE and FMS_CSRF_TOKEN are read from
 *   Apps Script > Project Settings > Script Properties.
 ************************************************************/

const FMS_CONFIG = {
  HOST: 'https://spx.shopee.co.id',

  CONVEYOR_ENDPOINT:
    '/api/wfm/admin/workstation/group/dropdown/list?count=50&fetch_all=1',

  PRODUCTIVITY_ENDPOINT:
    '/api/wfm/admin/workstation/productivity/productivity_individual_list',

  SOC_ENDPOINT:
    '/mgmt/api/pc/forward/data/api_mart/mgmt_app/data_api/' +
    'operation__soc_facility__operation_progress__10m_v4',

  ACTIVITY_TYPE: 12,
  DEFAULT_COUNT: 500,
  TOP_LIMIT: 10,
  BOTTOM_LIMIT: 10,
  AUTO_REFRESH_SECONDS: 30
};


/* ============================================================
   WEB APP
   ============================================================ */

function doGet() {
  return HtmlService
    .createHtmlOutputFromFile('index')
    .setTitle('Kartawijaya Monitoring')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}


/* ============================================================
   AUTH / SCRIPT PROPERTIES
   ============================================================ */

function getFmsProperties_() {
  const props =
    PropertiesService
      .getScriptProperties()
      .getProperties();

  return {
    cookie: String(props.FMS_COOKIE || '').trim(),
    csrf: String(props.FMS_CSRF_TOKEN || '').trim()
  };
}


function getFmsSessionStatus() {
  const auth = getFmsProperties_();

  return {
    success: true,
    checkedAt: Utilities.formatDate(
      new Date(),
      Session.getScriptTimeZone(),
      'yyyy-MM-dd HH:mm:ss'
    ),
    cookieStored: auth.cookie.length > 0,
    cookieLength: auth.cookie.length,
    csrfStored: auth.csrf.length > 0,
    csrfLength: auth.csrf.length
  };
}


/* ============================================================
   FMS HEADERS
   ============================================================ */

function getFmsHeaders_() {
  const auth = getFmsProperties_();

  if (!auth.cookie) {
    throw new Error(
      'FMS_COOKIE belum tersedia di Script Properties.'
    );
  }

  const headers = {
    'Accept': 'application/json, text/plain, */*',
    'Cookie': auth.cookie,
    'User-Agent':
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ' +
      'AppleWebKit/537.36 (KHTML, like Gecko) ' +
      'Chrome/151 Safari/537.36',
    'Referer': 'https://spx.shopee.co.id/',
    'Origin': 'https://spx.shopee.co.id'
  };

  if (auth.csrf) {
    headers['X-CsrfToken'] = auth.csrf;
  }

  return headers;
}


/* ============================================================
   HTTP GET
   ============================================================ */

function fmsGet_(url) {
  const response = UrlFetchApp.fetch(url, {
    method: 'get',
    headers: getFmsHeaders_(),
    muteHttpExceptions: true,
    followRedirects: true
  });

  const status = response.getResponseCode();
  const text = response.getContentText();

  console.log('FMS GET ' + status + ' | ' + url);

  if (status < 200 || status >= 300) {
    throw new Error(
      'FMS API ERROR HTTP ' + status + '\n' +
      text.substring(0, 1500)
    );
  }

  if (!text || !text.trim()) {
    throw new Error('FMS response kosong.\nURL: ' + url);
  }

  let json;

  try {
    json = JSON.parse(text);
  } catch (err) {
    throw new Error(
      'FMS response bukan JSON.\n' +
      text.substring(0, 1500)
    );
  }

  if (
    json &&
    json.retcode !== undefined &&
    Number(json.retcode) !== 0
  ) {
    throw new Error(
      'FMS API ERROR: ' +
      String(json.message || 'Unknown FMS error')
    );
  }

  return json;
}


/* ============================================================
   HTTP POST — SOC
   ============================================================ */

function fmsPostSoc_(url) {
  const response = UrlFetchApp.fetch(url, {
    method: 'post',
    headers: getFmsHeaders_(),
    payload: '',
    muteHttpExceptions: true,
    followRedirects: true
  });

  const status = response.getResponseCode();
  const text = response.getContentText();

  console.log('FMS SOC HTTP ' + status + ' | ' + url);
  console.log('FMS SOC RESPONSE LENGTH: ' + text.length);

  if (status < 200 || status >= 300) {
    throw new Error(
      'SOC API ERROR HTTP ' + status + '\n' +
      text.substring(0, 1500)
    );
  }

  if (!text || !text.trim()) {
    throw new Error(
      'SOC RESPONSE KOSONG\nHTTP: ' + status
    );
  }

  try {
    return JSON.parse(text);
  } catch (err) {
    throw new Error(
      'SOC RESPONSE BUKAN JSON\n' +
      text.substring(0, 1500)
    );
  }
}


/* ============================================================
   CONVEYOR 1-7
   ============================================================ */

function getConveyorList() {
  const url =
    FMS_CONFIG.HOST +
    FMS_CONFIG.CONVEYOR_ENDPOINT;

  const response = fmsGet_(url);
  const list = extractList_(response);

  const conveyors = [];

  list.forEach(function(item) {
    if (!item || typeof item !== 'object') return;

    const name = String(
      item.workstation_group_name ||
      item.workstationGroupName ||
      item.name ||
      ''
    ).trim();

    const match = name.match(/Conveyor\s*(\d+)/i);
    if (!match) return;

    const number = Number(match[1]);
    if (number < 1 || number > 7) return;

    const id = toNumber_(
      item.workstation_group_id !== undefined
        ? item.workstation_group_id
        : item.id
    );

    conveyors.push({
      conveyor: number,
      number: number,
      name: name || ('Conveyor ' + number),
      id: id,
      bizId: String(
        item.biz_workstation_group_id ||
        item.bizId ||
        ''
      )
    });
  });

  conveyors.sort(function(a, b) {
    return a.conveyor - b.conveyor;
  });

  const unique = [];
  const used = {};

  conveyors.forEach(function(item) {
    if (!used[item.conveyor]) {
      used[item.conveyor] = true;
      unique.push(item);
    }
  });

  console.log(
    'CONVEYOR FOUND: ' +
    JSON.stringify(unique)
  );

  return unique;
}


/* ============================================================
   DATE / SHIFT
   ============================================================ */

function parseDate_(dateString, timeString) {
  if (!dateString) {
    throw new Error('Tanggal belum dipilih.');
  }

  const p = String(dateString).split('-');

  if (p.length !== 3) {
    throw new Error(
      'Format tanggal tidak valid: ' + dateString
    );
  }

  const t =
    String(timeString || '00:00').split(':');

  return new Date(
    Number(p[0]),
    Number(p[1]) - 1,
    Number(p[2]),
    Number(t[0] || 0),
    Number(t[1] || 0),
    0,
    0
  );
}


function getShiftRange_(dateString, shift) {
  const s =
    String(shift || 'all').toLowerCase();

  if (s === '1') {
    return {
      start: parseDate_(dateString, '00:00'),
      end: parseDate_(dateString, '08:00')
    };
  }

  if (s === '2') {
    return {
      start: parseDate_(dateString, '08:00'),
      end: parseDate_(dateString, '16:00')
    };
  }

  if (s === '3') {
    return {
      start: parseDate_(dateString, '16:00'),
      end: parseDate_(dateString, '24:00')
    };
  }

  return {
    start: parseDate_(dateString, '00:00'),
    end: parseDate_(dateString, '24:00')
  };
}


function toUnixSeconds_(date) {
  return Math.floor(date.getTime() / 1000);
}


function pad2_(n) {
  return String(n).padStart(2, '0');
}


/* ============================================================
   PRODUCTIVITY URL
   ============================================================ */

function buildProductivityUrl_(
  conveyor,
  startTime,
  endTime
) {
  const params = {
    pageno: 1,
    count: FMS_CONFIG.DEFAULT_COUNT,
    workstation_group_id: conveyor.id,
    activity_type: FMS_CONFIG.ACTIVITY_TYPE,
    start_time: toUnixSeconds_(startTime),
    end_time: toUnixSeconds_(endTime)
  };

  const query = Object.keys(params)
    .map(function(key) {
      return (
        encodeURIComponent(key) +
        '=' +
        encodeURIComponent(params[key])
      );
    })
    .join('&');

  return (
    FMS_CONFIG.HOST +
    FMS_CONFIG.PRODUCTIVITY_ENDPOINT +
    '?' +
    query
  );
}


/* ============================================================
   VALUE HELPERS
   ============================================================ */

function firstDefined_(obj, names, fallback) {
  if (!obj || typeof obj !== 'object') {
    return fallback;
  }

  for (let i = 0; i < names.length; i++) {
    const key = names[i];

    if (
      obj[key] !== undefined &&
      obj[key] !== null &&
      obj[key] !== ''
    ) {
      return obj[key];
    }
  }

  return fallback;
}


function toNumber_(value) {
  if (
    value === null ||
    value === undefined ||
    value === ''
  ) {
    return 0;
  }

  if (typeof value === 'number') {
    return isFinite(value) ? value : 0;
  }

  const n =
    Number(
      String(value)
        .trim()
        .replace(/,/g, '')
    );

  return isFinite(n) ? n : 0;
}


function parseWorkingHours_(value) {
  if (
    value === null ||
    value === undefined ||
    value === ''
  ) {
    return 0;
  }

  if (typeof value === 'number') {
    return isFinite(value) ? value : 0;
  }

  const text = String(value).trim();

  if (!text) return 0;

  let m =
    text.match(/^(\d+):(\d{1,2}):(\d{1,2})$/);

  if (m) {
    return (
      Number(m[1]) +
      Number(m[2]) / 60 +
      Number(m[3]) / 3600
    );
  }

  m =
    text.match(/^(\d+):(\d{1,2})$/);

  if (m) {
    return (
      Number(m[1]) +
      Number(m[2]) / 60
    );
  }

  return toNumber_(text);
}


/* ============================================================
   FLEXIBLE RESPONSE LIST
   ============================================================ */

function extractList_(json) {
  if (!json || typeof json !== 'object') {
    return [];
  }

  const candidates = [
    json.list,
    json.data && json.data.list,
    json.data && json.data.data,
    json.data && json.data.items,
    json.data && json.data.rows,
    json.result && json.result.list,
    json.result && json.result.data,
    json.result && json.result.items
  ];

  for (let i = 0; i < candidates.length; i++) {
    if (Array.isArray(candidates[i])) {
      return candidates[i];
    }
  }

  return findFirstObjectArray_(json);
}


function findFirstObjectArray_(obj, depth) {
  depth = depth || 0;

  if (
    depth > 8 ||
    !obj ||
    typeof obj !== 'object'
  ) {
    return [];
  }

  if (Array.isArray(obj)) {
    if (
      obj.length === 0 ||
      typeof obj[0] === 'object'
    ) {
      return obj;
    }

    return [];
  }

  const keys = Object.keys(obj);

  for (let i = 0; i < keys.length; i++) {
    const value = obj[keys[i]];

    if (Array.isArray(value)) {
      if (
        value.length === 0 ||
        typeof value[0] === 'object'
      ) {
        return value;
      }
    }
  }

  for (let i = 0; i < keys.length; i++) {
    const value = obj[keys[i]];

    if (
      value &&
      typeof value === 'object' &&
      !Array.isArray(value)
    ) {
      const found =
        findFirstObjectArray_(
          value,
          depth + 1
        );

      if (found.length) {
        return found;
      }
    }
  }

  return [];
}


/* ============================================================
   NORMALIZE PRODUCTIVITY ROW
   ============================================================ */

function normalizeFmsRow_(item, conveyor) {
  item = item || {};

  const operator =
    firstDefined_(
      item,
      [
        'ops',
        'operator',
        'operator_name',
        'ops_id',
        'opsId',
        'user_name',
        'username'
      ],
      ''
    );

  const workstation =
    firstDefined_(
      item,
      [
        'workstation',
        'workstation_name',
        'workstationName'
      ],
      ''
    );

  let workingHours =
    parseWorkingHours_(
      firstDefined_(
        item,
        [
          'working_hours',
          'workingHours',
          'working_hour',
          'work_hours',
          'workHours',
          'total_working_hours'
        ],
        0
      )
    );

  const hourlyProductivity =
    toNumber_(
      firstDefined_(
        item,
        [
          'hourly_productivity',
          'hourlyProductivity',
          'productivity',
          'productivity_hourly'
        ],
        0
      )
    );

  const totalThroughput =
    toNumber_(
      firstDefined_(
        item,
        [
          'total_throughput',
          'totalThroughput',
          'throughput',
          'total_order_qty',
          'order_qty'
        ],
        0
      )
    );

  if (
    workingHours <= 0 &&
    totalThroughput > 0 &&
    hourlyProductivity > 0
  ) {
    workingHours =
      totalThroughput /
      hourlyProductivity;
  }

  return {
    conveyor: Number(conveyor.conveyor),
    conveyorName: conveyor.name || '',
    conveyorId: conveyor.id || '',
    bizId: conveyor.bizId || '',

    operator: String(operator).trim(),
    workstation: String(workstation).trim(),

    workstationGroup:
      String(
        firstDefined_(
          item,
          [
            'workstation_group',
            'workstationGroup',
            'workstation_group_name'
          ],
          ''
        )
      ).trim(),

    workingHours: workingHours,
    hourlyProductivity: hourlyProductivity,
    totalThroughput: totalThroughput,

    checkIn:
      firstDefined_(
        item,
        [
          'check_in_time',
          'checkIn',
          'check_in'
        ],
        ''
      ),

    checkOut:
      firstDefined_(
        item,
        [
          'check_out_time',
          'checkOut',
          'check_out'
        ],
        ''
      ),

    activityType:
      firstDefined_(
        item,
        [
          'activity_type',
          'activityType'
        ],
        0
      )
  };
}


/* ============================================================
   FMS DATETIME
   ============================================================ */

function parseFmsDateTime_(value) {
  if (!value) return null;

  if (
    Object.prototype.toString.call(value) ===
    '[object Date]'
  ) {
    return isNaN(value.getTime())
      ? null
      : new Date(value);
  }

  const text = String(value).trim();

  if (!text) return null;

  const first =
    text.split(';')[0].trim();

  let m =
    first.match(
      /^(\d{4})-(\d{2})-(\d{2})[ T](\d{2}):(\d{2})(?::(\d{2}))?/
    );

  if (m) {
    return new Date(
      Number(m[1]),
      Number(m[2]) - 1,
      Number(m[3]),
      Number(m[4]),
      Number(m[5]),
      Number(m[6] || 0),
      0
    );
  }

  const parsed = new Date(first);

  return isNaN(parsed.getTime())
    ? null
    : parsed;
}


function getFirstDateTime_(value) {
  if (!value) return null;

  const parts =
    String(value)
      .split(';')
      .map(function(x) {
        return x.trim();
      })
      .filter(Boolean);

  if (!parts.length) return null;

  return parseFmsDateTime_(parts[0]);
}


function getLastDateTime_(value) {
  if (!value) return null;

  const parts =
    String(value)
      .split(';')
      .map(function(x) {
        return x.trim();
      })
      .filter(Boolean);

  if (!parts.length) return null;

  return parseFmsDateTime_(
    parts[parts.length - 1]
  );
}


/* ============================================================
   OPERATOR AKTIF PER JAM
   ============================================================ */

function isOperatorActiveInHour_(
  row,
  hourStart,
  hourEnd
) {
  const checkIn =
    getFirstDateTime_(row.checkIn);

  const checkOut =
    getLastDateTime_(row.checkOut);

  if (!checkIn) return false;

  if (checkIn >= hourEnd) return false;

  if (
    checkOut &&
    checkOut <= hourStart
  ) {
    return false;
  }

  return true;
}


function operatorKey_(conveyor, operator) {
  return (
    String(conveyor) +
    '|' +
    String(operator || '')
      .trim()
      .toLowerCase()
  );
}


/* ============================================================
   HOURLY REQUESTS
   ============================================================ */

function buildHourlyRequests_(
  dateString,
  range,
  conveyors,
  selectedConveyor
) {
  const requests = [];

  const baseDate =
    parseDate_(dateString, '00:00');

  for (let h = 0; h < 24; h++) {
    const hourStart =
      new Date(baseDate.getTime());

    hourStart.setHours(h, 0, 0, 0);

    const hourEnd =
      new Date(hourStart.getTime());

    hourEnd.setHours(
      h + 1,
      0,
      0,
      0
    );

    if (
      hourEnd <= range.start ||
      hourStart >= range.end
    ) {
      continue;
    }

    conveyors.forEach(function(conveyor) {
      if (
        String(selectedConveyor || 'all') !== 'all' &&
        String(conveyor.conveyor) !==
          String(selectedConveyor)
      ) {
        return;
      }

      requests.push({
        hour: h,
        conveyor: conveyor,

        url:
          buildProductivityUrl_(
            conveyor,
            hourStart,
            hourEnd
          ),

        method: 'get',
        headers: getFmsHeaders_(),
        muteHttpExceptions: true,
        followRedirects: true
      });
    });
  }

  return requests;
}


/* ============================================================
   HOURLY AGGREGATION
   ============================================================ */

function aggregateHourly_(
  baseDate,
  byHour
) {
  const hourly = [];

  for (let h = 0; h < 24; h++) {
    const hourStart =
      new Date(baseDate.getTime());

    hourStart.setHours(
      h,
      0,
      0,
      0
    );

    const hourEnd =
      new Date(hourStart.getTime());

    hourEnd.setHours(
      h + 1,
      0,
      0,
      0
    );

    const rows =
      byHour[h] || [];

    let throughput = 0;
    let workingHours = 0;

    const operatorSet =
      new Set();

    const conveyorMap = {};

    rows.forEach(function(row) {
      throughput +=
        Number(
          row.totalThroughput || 0
        );

      workingHours +=
        Number(
          row.workingHours || 0
        );

      if (
        row.operator &&
        isOperatorActiveInHour_(
          row,
          hourStart,
          hourEnd
        )
      ) {
        operatorSet.add(
          operatorKey_(
            row.conveyor,
            row.operator
          )
        );
      }

      const c =
        Number(row.conveyor);

      if (!conveyorMap[c]) {
        conveyorMap[c] = {
          conveyor: c,
          throughput: 0,
          workingHours: 0,
          operators: new Set()
        };
      }

      conveyorMap[c].throughput +=
        Number(
          row.totalThroughput || 0
        );

      conveyorMap[c].workingHours +=
        Number(
          row.workingHours || 0
        );

      if (
        row.operator &&
        isOperatorActiveInHour_(
          row,
          hourStart,
          hourEnd
        )
      ) {
        conveyorMap[c].operators.add(
          operatorKey_(
            c,
            row.operator
          )
        );
      }
    });

    const conveyorRows = [];

    for (let c = 1; c <= 7; c++) {
      const item =
        conveyorMap[c];

      if (!item) {
        conveyorRows.push({
          conveyor: c,
          throughput: 0,
          workingHours: 0,
          productivity: 0,
          mp: 0
        });
        continue;
      }

      conveyorRows.push({
        conveyor: c,
        throughput: item.throughput,
        workingHours: item.workingHours,

        productivity:
          item.workingHours > 0
            ? item.throughput /
              item.workingHours
            : 0,

        mp:
          item.operators.size
      });
    }

    hourly.push({
      hour: h,

      label:
        pad2_(h) +
        ':00 - ' +
        pad2_(h) +
        ':59',

      throughput: throughput,
      workingHours: workingHours,

      productivity:
        workingHours > 0
          ? throughput /
            workingHours
          : 0,

      mp:
        operatorSet.size,

      conveyors:
        conveyorRows
    });
  }

  return hourly;
}


/* ============================================================
   TOP / BOTTOM 10
   ============================================================ */

function buildOperatorRanking_(rows) {
  const map = {};

  rows.forEach(function(row) {
    const operator =
      String(
        row.operator || ''
      ).trim();

    if (!operator) return;

    const key =
      operatorKey_(
        row.conveyor,
        operator
      );

    if (!map[key]) {
      map[key] = {
        operator: operator,
        conveyor:
          Number(row.conveyor),
        workstation:
          row.workstation || '',
        throughput: 0,
        workingHours: 0,
        checkIn:
          row.checkIn || '',
        checkOut:
          row.checkOut || ''
      };
    }

    map[key].throughput +=
      Number(
        row.totalThroughput || 0
      );

    let wh =
      Number(
        row.workingHours || 0
      );

    if (
      wh <= 0 &&
      Number(row.totalThroughput || 0) > 0 &&
      Number(row.hourlyProductivity || 0) > 0
    ) {
      wh =
        Number(row.totalThroughput) /
        Number(row.hourlyProductivity);
    }

    map[key].workingHours += wh;

    if (
      !map[key].checkIn &&
      row.checkIn
    ) {
      map[key].checkIn =
        row.checkIn;
    }

    if (row.checkOut) {
      map[key].checkOut =
        row.checkOut;
    }
  });

  const all =
    Object.keys(map)
      .map(function(key) {
        const x = map[key];

        const productivity =
          x.workingHours > 0
            ? x.throughput /
              x.workingHours
            : 0;

        return {
          operator: x.operator,
          conveyor: x.conveyor,
          workstation: x.workstation,
          workingHours:
            x.workingHours,
          throughput:
            x.throughput,
          productivity:
            productivity,
          checkIn:
            x.checkIn,
          checkOut:
            x.checkOut
        };
      })
      .filter(function(x) {
        return (
          x.operator &&
          (
            x.throughput > 0 ||
            x.workingHours > 0
          )
        );
      });

  const top10 =
    all.slice()
      .sort(function(a, b) {
        return (
          b.productivity -
          a.productivity
        ) || (
          b.throughput -
          a.throughput
        );
      })
      .slice(
        0,
        FMS_CONFIG.TOP_LIMIT
      )
      .map(function(x, i) {
        x.rank = i + 1;
        return x;
      });

  const bottom10 =
    all.slice()
      .sort(function(a, b) {
        return (
          a.productivity -
          b.productivity
        ) || (
          a.throughput -
          b.throughput
        );
      })
      .slice(
        0,
        FMS_CONFIG.BOTTOM_LIMIT
      )
      .map(function(x, i) {
        x.rank = i + 1;
        return x;
      });

  return {
    all: all,
    top10: top10,
    bottom10: bottom10
  };
}


/* ============================================================
   SOC
   ============================================================ */

function getSocData_() {
  const url =
    FMS_CONFIG.HOST +
    FMS_CONFIG.SOC_ENDPOINT;

  console.log('================================');
  console.log('FMS SOC REQUEST');
  console.log('URL: ' + url);
  console.log('METHOD: POST');
  console.log('BODY: EMPTY');
  console.log('================================');

  try {
    const json =
      fmsPostSoc_(url);

    const data =
      json && json.data
        ? json.data
        : json;

    const soc =
      toNumber_(
        data &&
        data.soc_in_system_order_qty_td
      );

    const transported =
      toNumber_(
        data &&
        data.soc_transported_in_system_order_qty_td
      );

    const socReceived =
      toNumber_(
        data &&
        data.soc_received_in_system_order_qty_td
      );

    console.log('SOC = ' + soc);
    console.log(
      'TRANSPORTED = ' +
      transported
    );
    console.log(
      'SOC RECEIVED = ' +
      socReceived
    );

    return {
      soc: soc,
      transported: transported,
      socReceived: socReceived,
      error: null
    };

  } catch (err) {
    console.log(
      'SOC request gagal: ' +
      err.message
    );

    return {
      soc: 0,
      transported: 0,
      socReceived: 0,
      error: err.message
    };
  }
}


/* ============================================================
   MAIN DASHBOARD
   ============================================================ */

function getDashboardData(params) {
  params = params || {};

  const date =
    params.date ||
    params.fromDate ||
    Utilities.formatDate(
      new Date(),
      Session.getScriptTimeZone(),
      'yyyy-MM-dd'
    );

  const shift =
    String(
      params.shift || 'all'
    ).toLowerCase();

  const conveyorFilter =
    String(
      params.conveyor || 'all'
    ).toLowerCase();

  const range =
    getShiftRange_(
      date,
      shift
    );

  console.log('====================================');
  console.log('FMS DASHBOARD');
  console.log('DATE: ' + date);
  console.log('SHIFT: ' + shift);
  console.log('START: ' + range.start);
  console.log('END: ' + range.end);

  const conveyors =
    getConveyorList();

  if (!conveyors.length) {
    throw new Error(
      'Conveyor 1-7 tidak ditemukan.'
    );
  }

  const byHour = {};

  for (let h = 0; h < 24; h++) {
    byHour[h] = [];
  }

  const requests =
    buildHourlyRequests_(
      date,
      range,
      conveyors,
      conveyorFilter
    );

  console.log(
    'TOTAL REQUEST: ' +
    requests.length
  );

  const allRows = [];

  if (requests.length) {
    const responses =
      UrlFetchApp.fetchAll(
        requests
      );

    responses.forEach(
      function(response, index) {
        const req =
          requests[index];

        const status =
          response.getResponseCode();

        const text =
          response.getContentText();

        if (
          status < 200 ||
          status >= 300
        ) {
          console.log(
            'GAGAL C' +
            req.conveyor.conveyor +
            ' H' +
            req.hour +
            ' HTTP ' +
            status
          );
          return;
        }

        if (!text || !text.trim()) {
          console.log(
            'KOSONG C' +
            req.conveyor.conveyor +
            ' H' +
            req.hour
          );
          return;
        }

        let json;

        try {
          json = JSON.parse(text);
        } catch (err) {
          console.log(
            'JSON ERROR C' +
            req.conveyor.conveyor +
            ' H' +
            req.hour
          );
          return;
        }

        if (
          json &&
          json.retcode !== undefined &&
          Number(json.retcode) !== 0
        ) {
          console.log(
            'FMS ERROR C' +
            req.conveyor.conveyor +
            ' H' +
            req.hour +
            ': ' +
            String(
              json.message || ''
            )
          );
          return;
        }

        const list =
          extractList_(json);

        list.forEach(
          function(item) {
            const row =
              normalizeFmsRow_(
                item,
                req.conveyor
              );

            row.requestHour =
              req.hour;

            byHour[
              req.hour
            ].push(row);

            allRows.push(row);
          }
        );
      }
    );
  }

  console.log(
    'TOTAL ROW: ' +
    allRows.length
  );

  const hourly =
    aggregateHourly_(
      parseDate_(date, '00:00'),
      byHour
    );

  const filteredHourly =
    hourly.filter(
      function(item) {
        const start =
          parseDate_(
            date,
            pad2_(item.hour) +
            ':00'
          );

        const end =
          new Date(
            start.getTime()
          );

        end.setHours(
          start.getHours() + 1
        );

        return (
          end > range.start &&
          start < range.end
        );
      }
    );

  /*
   * Ranking memakai row dari jam yang masuk shift.
   */
  const rankingRows =
    allRows.filter(
      function(row) {
        const h =
          Number(row.requestHour);

        if (
          range.end.getDate() !==
          range.start.getDate()
        ) {
          return (
            h >= range.start.getHours() &&
            h < 24
          );
        }

        return (
          h >= range.start.getHours() &&
          h < range.end.getHours()
        );
      }
    );

  const ranking =
    buildOperatorRanking_(
      rankingRows
    );

  let totalThroughput = 0;
  let totalWorkingHours = 0;

  ranking.all.forEach(
    function(row) {
      totalThroughput +=
        Number(
          row.throughput || 0
        );

      totalWorkingHours +=
        Number(
          row.workingHours || 0
        );
    }
  );

  const totalProductivity =
    totalWorkingHours > 0
      ? totalThroughput /
        totalWorkingHours
      : 0;

  /*
   * MP aktif saat ini.
   */
  const now = new Date();

  const todayString =
    Utilities.formatDate(
      now,
      Session.getScriptTimeZone(),
      'yyyy-MM-dd'
    );

  const currentHour =
    todayString === date
      ? now.getHours()
      : -1;

  const currentRows =
    currentHour >= 0
      ? byHour[currentHour] || []
      : [];

  const currentHourStart =
    currentHour >= 0
      ? parseDate_(
          date,
          pad2_(currentHour) +
          ':00'
        )
      : null;

  const currentHourEnd =
    currentHourStart
      ? new Date(
          currentHourStart.getTime()
        )
      : null;

  if (currentHourEnd) {
    currentHourEnd.setHours(
      currentHourEnd.getHours() + 1
    );
  }

  const activeNowSet =
    new Set();

  if (
    currentHourStart &&
    currentHourEnd
  ) {
    currentRows.forEach(
      function(row) {
        if (
          isOperatorActiveInHour_(
            row,
            currentHourStart,
            currentHourEnd
          )
        ) {
          activeNowSet.add(
            operatorKey_(
              row.conveyor,
              row.operator
            )
          );
        }
      }
    );
  }

  /*
   * Conveyor summary.
   */
  const conveyorSummary = [];

  for (let c = 1; c <= 7; c++) {
    let throughput = 0;
    let workingHours = 0;
    let mp = 0;

    filteredHourly.forEach(
      function(hour) {
        const item =
          hour.conveyors.find(
            function(x) {
              return (
                Number(x.conveyor) === c
              );
            }
          );

        if (!item) return;

        throughput +=
          Number(
            item.throughput || 0
          );

        workingHours +=
          Number(
            item.workingHours || 0
          );

        mp = Math.max(
          mp,
          Number(item.mp || 0)
        );
      }
    );

    const info =
      conveyors.find(
        function(x) {
          return (
            Number(x.conveyor) === c
          );
        }
      ) || {};

    conveyorSummary.push({
      conveyor: c,
      number: c,
      name:
        info.name ||
        'Conveyor ' + c,
      id:
        info.id || '',
      bizId:
        info.bizId || '',

      throughput:
        round_(
          throughput,
          0
        ),

      workingHours:
        round_(
          workingHours,
          2
        ),

      productivity:
        round_(
          workingHours > 0
            ? throughput /
              workingHours
            : 0,
          0
        ),

      operators: mp,

      active:
        throughput > 0 ||
        mp > 0
    });
  }

  const activeConveyor =
    conveyorSummary.filter(
      function(x) {
        return x.active;
      }
    ).length;

  const shiftSummary = {
    shift1:
      summarizeHours_(
        hourly,
        0,
        8
      ),

    shift2:
      summarizeHours_(
        hourly,
        8,
        16
      ),

    shift3:
      summarizeHours_(
        hourly,
        16,
        24
      )
  };

  /*
   * SOC tetap menggunakan POST endpoint yang sudah terbukti
   * mengembalikan JSON.
   */
  const socData =
    getSocData_();

  return {
    success: true,

    mode:
      String(
        params.mode || 'LIVE'
      ).toUpperCase(),

    generatedAt:
      new Date().toISOString(),

    date: date,
    shift: shift,

    range: {
      start:
        range.start.toISOString(),
      end:
        range.end.toISOString()
    },

    conveyorList:
      conveyors,

    summary: {
      totalThroughput:
        round_(
          totalThroughput,
          0
        ),

      totalWorkingHours:
        round_(
          totalWorkingHours,
          2
        ),

      productivity:
        round_(
          totalProductivity,
          0
        ),

      totalOperator:
        ranking.all.length,

      activeOperatorNow:
        activeNowSet.size,

      activeConveyor:
        activeConveyor
    },

    soc:
      socData,

    conveyors:
      conveyorSummary,

    hourly:
      filteredHourly,

    shiftSummary:
      shiftSummary,

    top10:
      ranking.top10,

    bottom10:
      ranking.bottom10,

    allOperators:
      ranking.all,

    source: {
      endpoint:
        FMS_CONFIG.PRODUCTIVITY_ENDPOINT,

      activityType:
        FMS_CONFIG.ACTIVITY_TYPE,

      requestCount:
        requests.length,

      rowCount:
        allRows.length
    }
  };
}


/* ============================================================
   SHIFT SUMMARY
   ============================================================ */

function summarizeHours_(
  hourly,
  startHour,
  endHour
) {
  let throughput = 0;
  let workingHours = 0;

  hourly.forEach(
    function(item) {
      if (
        item.hour >= startHour &&
        item.hour < endHour
      ) {
        throughput +=
          Number(
            item.throughput || 0
          );

        workingHours +=
          Number(
            item.workingHours || 0
          );
      }
    }
  );

  return {
    throughput:
      round_(
        throughput,
        0
      ),

    workingHours:
      round_(
        workingHours,
        2
      ),

    productivity:
      round_(
        workingHours > 0
          ? throughput /
            workingHours
          : 0,
        0
      )
  };
}


/* ============================================================
   TEST
   ============================================================ */

function testFMS() {
  const result =
    getConveyorList();

  console.log(
    JSON.stringify(
      result,
      null,
      2
    )
  );

  return result;
}


function testSoc() {
  const result =
    getSocData_();

  console.log(
    JSON.stringify(
      result,
      null,
      2
    )
  );

  return result;
}


function testDashboard() {
  const date =
    Utilities.formatDate(
      new Date(),
      Session.getScriptTimeZone(),
      'yyyy-MM-dd'
    );

  const result =
    getDashboardData({
      mode: 'LIVE',
      date: date,
      shift: 'all',
      conveyor: 'all'
    });

  console.log(
    JSON.stringify(
      result,
      null,
      2
    )
  );

  return result;
}


/* ============================================================
   ROUND
   ============================================================ */

function round_(value, digits) {
  const p =
    Math.pow(
      10,
      Number(digits || 0)
    );

  return (
    Math.round(
      Number(value || 0) * p
    ) / p
  );
}

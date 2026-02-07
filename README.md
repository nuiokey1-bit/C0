[Uploading Index.HTML…]()
import React, { useState, useEffect } from 'react';
import { Upload, FileText, AlertCircle, CheckCircle, User, Users, Calendar, Table, Check, Download, Layers } from 'lucide-react';

export default function App() {
  const [workbook, setWorkbook] = useState(null);
  const [sheetNames, setSheetNames] = useState([]);
  const [selectedSheet, setSelectedSheet] = useState(null);
  const [data, setData] = useState(null);
  const [globalData, setGlobalData] = useState(null); // New state for aggregated data
  const [isScriptLoaded, setIsScriptLoaded] = useState(false);
  const [processingStatus, setProcessingStatus] = useState('');

  // Load SheetJS
  useEffect(() => {
    const script = document.createElement('script');
    script.src = "https://cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js";
    script.async = true;
    script.onload = () => setIsScriptLoaded(true);
    document.body.appendChild(script);
    return () => {
      document.body.removeChild(script);
    }
  }, []);

  const getThaiDate = (day, month, yearAD) => {
    const date = new Date(yearAD, month - 1, day);
    const days = ["อาทิตย์", "จันทร์", "อังคาร", "พุธ", "พฤหัสบดี", "ศุกร์", "เสาร์"];
    const months = [
        "มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน", "พฤษภาคม", "มิถุนายน",
        "กรกฎาคม", "สิงหาคม", "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"
    ];
    
    const dayName = days[date.getDay()];
    const monthName = months[date.getMonth()];
    const yearBE = yearAD + 543;

    return `${dayName}ที่ ${day} ${monthName} ${yearBE}`;
  };

  const handleFileUpload = (event) => {
    const file = event.target.files[0];
    if (!file) return;

    if (!isScriptLoaded) {
      alert("กำลังโหลดไลบรารี Excel กรุณารอสักครู่...");
      return;
    }

    setProcessingStatus('กำลังอ่านไฟล์...');
    setData(null);
    setGlobalData(null);
    setSheetNames([]);
    setSelectedSheet(null);

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = new Uint8Array(e.target.result);
        const wb = window.XLSX.read(data, { type: 'array' });
        
        setWorkbook(wb);
        setSheetNames(wb.SheetNames);
        
        // 1. Generate Global Summary from ALL sheets
        generateGlobalSummary(wb);

        // 2. Auto-select "C0 ม.ค." if exists, or first sheet for detailed view
        const c0Sheet = wb.SheetNames.find(s => s.includes("C0") || s.includes("C-0"));
        if (c0Sheet) {
            handleSheetSelect(c0Sheet, wb);
        } else if (wb.SheetNames.length === 1) {
            handleSheetSelect(wb.SheetNames[0], wb);
        }
        
        setProcessingStatus('');

      } catch (err) {
        console.error(err);
        alert("เกิดข้อผิดพลาดในการอ่านไฟล์ Excel: " + err.message);
        setProcessingStatus('');
      }
    };
    reader.readAsArrayBuffer(file);
  };

  // New function to aggregate data from all sheets
  const generateGlobalSummary = (wb) => {
    const personMap = new Map();

    wb.SheetNames.forEach(sheetName => {
        const ws = wb.Sheets[sheetName];
        const rows = window.XLSX.utils.sheet_to_json(ws, { header: 1, defval: "" });
        const { processed } = processSheetData(rows, sheetName); // Reuse existing logic

        processed.forEach(p => {
            if (personMap.has(p.name)) {
                const existing = personMap.get(p.name);
                existing.total += p.total;
                // Merge dates, adding source sheet info
                const newDates = p.dates.map(d => ({ ...d, source: sheetName }));
                existing.dates = [...existing.dates, ...newDates];
                // Keep the first valid position/line found
                if (!existing.position && p.position) existing.position = p.position;
                if (!existing.line && p.line) existing.line = p.line;
            } else {
                 const newDates = p.dates.map(d => ({ ...d, source: sheetName }));
                 personMap.set(p.name, { ...p, dates: newDates });
            }
        });
    });

    const combinedList = Array.from(personMap.values());
    combinedList.sort((a, b) => b.total - a.total);
    setGlobalData(combinedList);
  };

  const handleSheetSelect = (sheetName, wbInstance = workbook) => {
    setSelectedSheet(sheetName);
    const worksheet = wbInstance.Sheets[sheetName];
    // Get raw rows including empty cells
    const rows = window.XLSX.utils.sheet_to_json(worksheet, { header: 1, defval: "" });
    const processedData = processSheetData(rows, sheetName);
    
    // Sort by Total desc
    processedData.processed.sort((a, b) => b.total - a.total);
    
    setData(processedData);
  };

  const processSheetData = (rows, sheetName) => {
    const processed = [];
    const noC0 = [];

    // Skip header rows (Assuming data starts at index 3 based on user file)
    const dataRows = rows.slice(3);

    dataRows.forEach(row => {
        const name = (row[1] || "").toString().trim();
        const position = (row[2] || "").toString().trim();
        const line = (row[3] || "").toString().trim();
        
        let calculatedTotal = 0;
        const dates = [];

        const isNameValid = name.toLowerCase() !== 'nan' && name !== '' && name !== 'ชื่อ-นามสกุล';
        if (!isNameValid) return;

        // Columns 4 to 34 are days 1-31 (Indices 4-34)
        for (let d = 4; d <= 34; d++) {
            const valRaw = (row[d] || "").toString().trim();
            const val = valRaw.toUpperCase(); 
            
            // STRICT FILTER: Only "1", "1.0", "C-0", or "A-5"
            if (val === "1" || val === "1.0" || val === "C-0" || val === "A-5") {
                const dayNum = d - 3; // Col 4 is Day 1
                dates.push({
                    raw: dayNum,
                    display: getThaiDate(dayNum, 1, 2026), 
                    note: val === "A-5" ? "(A-5)" : (val === "C-0" ? "(C-0)" : "")
                });
                calculatedTotal++;
            }
        }

        if (calculatedTotal > 0) {
            processed.push({ name, position, line, total: calculatedTotal, dates });
        } else {
             if (position || line) {
                noC0.push({ name, position, line });
             }
        }
    });

    return { processed, noC0 };
  };

  return (
    <div className="min-h-screen bg-gray-100 py-8 px-4 font-sans text-gray-800">
      <div className="max-w-4xl mx-auto bg-white shadow-lg rounded-xl overflow-hidden min-h-[800px]">
        
        {/* Header Section */}
        <div className="bg-blue-600 text-white p-6 text-center">
          <h1 className="text-2xl font-bold mb-2 flex justify-center items-center gap-2">
            📊 ระบบวิเคราะห์วัน C-0 และ A-5
          </h1>
          <p className="text-blue-100 opacity-90">รวมข้อมูลจากทุกชีต และดูรายละเอียดรายบุคคล</p>
        </div>

        {/* Controls */}
        <div className="p-6 border-b border-gray-200 bg-gray-50">
            <div className="flex flex-col md:flex-row gap-4 items-center justify-center mb-4">
                <label className="cursor-pointer bg-white border border-gray-300 text-gray-700 px-4 py-2 rounded shadow-sm hover:bg-gray-50 flex items-center gap-2">
                    <Upload className="w-4 h-4" />
                    {processingStatus ? processingStatus : "อัปโหลดไฟล์ Excel"}
                    <input 
                        type="file" 
                        accept=".xlsx, .xls, .csv"
                        onChange={handleFileUpload}
                        className="hidden" 
                        disabled={!isScriptLoaded}
                    />
                </label>
                
                {/* Sheet Selector */}
                {sheetNames.length > 0 && (
                    <div className="flex items-center gap-2 overflow-x-auto max-w-full pb-2 md:pb-0">
                        <span className="text-sm text-gray-500 whitespace-nowrap">เลือกชีต:</span>
                        {sheetNames.map(sheet => (
                            <button
                                key={sheet}
                                onClick={() => handleSheetSelect(sheet)}
                                className={`px-3 py-1 rounded-full text-xs font-medium whitespace-nowrap transition-colors
                                    ${selectedSheet === sheet 
                                        ? 'bg-blue-600 text-white' 
                                        : 'bg-gray-200 text-gray-600 hover:bg-gray-300'
                                    }`}
                            >
                                {sheet}
                            </button>
                        ))}
                    </div>
                )}
            </div>
        </div>

        <div className="p-8">
            
            {/* Global Summary Card */}
            {globalData && globalData.length > 0 && (
                <div className="mb-10 bg-gradient-to-r from-blue-50 to-indigo-50 border border-blue-200 rounded-xl p-6 shadow-sm">
                    <h2 className="text-xl font-bold mb-4 flex items-center gap-2 text-blue-800">
                        <Layers className="w-6 h-6" />
                        ภาพรวมจากทุกชีต (รวม {globalData.reduce((sum, p) => sum + p.total, 0)} วัน)
                    </h2>
                    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
                        {globalData.map((person, idx) => (
                            <div key={idx} className="bg-white p-3 rounded-lg border border-blue-100 shadow-sm flex items-center justify-between">
                                <div>
                                    <div className="font-bold text-gray-800 text-sm">{idx + 1}. {person.name}</div>
                                    <div className="text-xs text-gray-500">{person.position}</div>
                                </div>
                                <div className="text-center">
                                    <span className="block text-lg font-bold text-blue-600">{person.total}</span>
                                    <span className="text-[10px] text-gray-400">วัน</span>
                                </div>
                            </div>
                        ))}
                    </div>
                </div>
            )}

            {/* Sheet Detail Content */}
            {data && selectedSheet ? (
                <div>
                    <div className="flex items-center gap-2 mb-6 pb-2 border-b border-gray-200">
                        <FileText className="w-5 h-5 text-gray-400" />
                        <span className="text-gray-500">รายละเอียดจากชีต:</span>
                        <span className="font-bold text-lg text-gray-800 bg-gray-100 px-3 py-1 rounded">{selectedSheet}</span>
                    </div>

                    {/* Groups */}
                    <ReportSection 
                        title="มี C-0 มากที่สุด (4 วัน):" 
                        items={data.processed.filter(p => p.total >= 4)} 
                        icon="🔴"
                        startIndex={1}
                    />
                    <ReportSection 
                        title="มี C-0 ปานกลาง (2-3 วัน):" 
                        items={data.processed.filter(p => p.total >= 2 && p.total < 4)} 
                        icon="🟡"
                        startIndex={data.processed.filter(p => p.total >= 4).length + 1}
                    />
                    <ReportSection 
                        title="มี C-0 น้อย (1 วัน):" 
                        items={data.processed.filter(p => p.total === 1)} 
                        icon="🟢"
                        startIndex={data.processed.filter(p => p.total >= 2).length + 1}
                    />

                    <hr className="my-8 border-gray-300" />

                    {/* Summary Table */}
                    <div className="mb-8">
                        <h2 className="text-xl font-bold mb-4 flex items-center gap-2">
                            📈 สรุป (เฉพาะชีตนี้)
                        </h2>
                        <div className="overflow-x-auto">
                            <table className="w-full border-collapse border border-gray-300 text-sm">
                                <thead>
                                    <tr className="bg-gray-100">
                                        <th className="border border-gray-300 px-4 py-2 text-left">ลำดับ</th>
                                        <th className="border border-gray-300 px-4 py-2 text-left">ชื่อ</th>
                                        <th className="border border-gray-300 px-4 py-2 text-left">ตำแหน่ง</th>
                                        <th className="border border-gray-300 px-4 py-2 text-left">สาย</th>
                                        <th className="border border-gray-300 px-4 py-2 text-left">จำนวนวัน</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    {data.processed.map((p, idx) => (
                                        <tr key={idx} className={idx % 2 === 0 ? 'bg-white' : 'bg-gray-50'}>
                                            <td className="border border-gray-300 px-4 py-2 text-center">{idx + 1}</td>
                                            <td className="border border-gray-300 px-4 py-2">{p.name}</td>
                                            <td className="border border-gray-300 px-4 py-2">{p.position}</td>
                                            <td className="border border-gray-300 px-4 py-2">{p.line}</td>
                                            <td className="border border-gray-300 px-4 py-2 font-bold text-center">{p.total} วัน</td>
                                        </tr>
                                    ))}
                                </tbody>
                                <tfoot>
                                    <tr className="bg-blue-50 font-bold">
                                        <td colSpan="4" className="border border-gray-300 px-4 py-2 text-right">รวมทั้งหมด:</td>
                                        <td className="border border-gray-300 px-4 py-2 text-center text-blue-700">
                                            {data.processed.reduce((sum, p) => sum + p.total, 0)} วัน
                                        </td>
                                    </tr>
                                </tfoot>
                            </table>
                        </div>
                    </div>

                    {/* No C-0 Section */}
                    {data.noC0.length > 0 && (
                        <div className="mt-8 p-6 bg-gray-50 rounded-xl border border-gray-200">
                            <h3 className="font-bold text-gray-500 mb-4 flex items-center gap-2">
                                <User className="w-5 h-5" />
                                ไม่พบข้อมูล C-0 หรือ A-5 ({data.noC0.length} คน):
                            </h3>
                            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-3">
                                {data.noC0.map((p, i) => (
                                    <div key={i} className="flex items-center gap-2 text-sm bg-white p-2 rounded border border-gray-100">
                                        <span className="text-gray-400 font-mono w-6 text-right">{i + 1}.</span>
                                        <div>
                                            <div className="font-medium text-gray-700">{p.name}</div>
                                            <div className="text-xs text-gray-400">{p.position} • {p.line}</div>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        </div>
                    )}
                </div>
            ) : (
                !globalData && (
                    <div className="flex flex-col items-center justify-center h-[400px] text-gray-400">
                        <FileText className="w-16 h-16 mb-4 opacity-20" />
                        <p>กรุณาอัปโหลดไฟล์ Excel เพื่อเริ่มการวิเคราะห์</p>
                    </div>
                )
            )}
        </div>
      </div>
    </div>
  );
}

const ReportSection = ({ title, items, icon, startIndex }) => {
    if (items.length === 0) return null;

    let currentIndex = startIndex;

    return (
        <div className="mb-6">
            <h2 className="text-lg font-bold mb-4 text-gray-800">
                {icon} {title}
            </h2>
            <div className="space-y-4">
                {items.map((person, idx) => (
                    <div key={idx} className="ml-2">
                        <div className="font-semibold text-gray-900 text-base">
                            {currentIndex++}. {person.name} - {person.position}, สาย{person.line}
                        </div>
                        <ul className="mt-1 ml-6 space-y-1">
                            {person.dates.map((d, i) => (
                                <li key={i} className="text-gray-600 text-sm">
                                    {d.display} {d.note && <span className="text-red-500 font-bold text-xs">{d.note}</span>}
                                    {/* Show source sheet if coming from global summary, but here we are in sheet specific view */}
                                    {d.source && <span className="text-gray-400 text-xs ml-2">({d.source})</span>}
                                </li>
                            ))}
                        </ul>
                    </div>
                ))}
            </div>
        </div>
    );
};

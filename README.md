https://registry.npmjs.org/exceljs/-/exceljs-4.4.0.tgz

https://registry.npmjs.org/exceljs/-/exceljs-4.4.0.tgz?utm_source=chatgpt.com

https://www.npmjs.com/package/exceljs?utm_source=chatgpt.com



const ExcelJS = require("exceljs");

/**
 * Generate an Excel file with last 4 months' data, each as a sheet
 * @param {Array} rows - Result rows from client.query()
 * @param {Array} headers - Array of pretty headers for Excel
 * @param {string} outputFile - Path for output Excel file
 */
async function generateExcelByLast4Months(rows, headers, outputFile = "Report.xlsx") {
  const workbook = new ExcelJS.Workbook();

  // Helper: build last 4 months
  const months = [];
  const today = new Date();
  let year = today.getFullYear();
  let month = today.getMonth() + 1; // 1–12

  for (let i = 0; i < 4; i++) {
    let m = month - i;
    let y = year;
    if (m <= 0) {
      m += 12;
      y -= 1;
    }
    months.push({
      key: `${y}-${m}`,
      name: `${new Date(y, m - 1).toLocaleString("default", { month: "long" })}-${y}`
    });
  }

  // Group rows by report_year + report_month
  const grouped = rows.reduce((acc, row) => {
    const key = `${row.report_year}-${row.report_month}`;
    if (!acc[key]) acc[key] = [];
    acc[key].push(row);
    return acc;
  }, {});

  // Build sheets
  for (const { key, name } of months) {
    const worksheet = workbook.addWorksheet(name);

    // Add header row
    worksheet.addRow(headers);

    // Add data rows (excluding first 2 columns: report_year, report_month)
    const data = grouped[key] || [];
    data.forEach(row => {
      const values = Object.values(row).slice(2);
      worksheet.addRow(values);
    });
  }

  // Write with compression
  await workbook.xlsx.writeFile(outputFile);
  console.log(`Excel file created: ${outputFile}`);
}

module.exports = { generateExcelByLast4Months };

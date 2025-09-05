const XLSX = require("xlsx");

/**
 * Generate an Excel file with last 4 months' data, each as a sheet
 * @param {Array} rows - Result rows from client.query()
 * @param {Array} headers - Array of pretty headers for Excel (excluding report_year, report_month)
 * @param {string} outputFile - Path for output Excel file
 */
function generateExcelByLast4Months(rows, headers, outputFile = "Report.xlsx") {
  const workbook = XLSX.utils.book_new();

  // Helper: build month keys for last 4 months
  const months = [];
  const today = new Date();
  for (let i = 0; i < 4; i++) {
    const d = new Date(today.getFullYear(), today.getMonth() - i, 1);
    months.push({
      key: `${d.getFullYear()}-${d.getMonth() + 1}`, // e.g. "2025-9"
      name: `${d.toLocaleString("default", { month: "long" })}-${d.getFullYear()}` // e.g. "September-2025"
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
  months.forEach(({ key, name }) => {
    const data = grouped[key] || [];

    // Exclude first two columns (report_year, report_month)
    const aoa = [headers];
    data.forEach(row => {
      const values = Object.values(row).slice(2); // drop year + month
      aoa.push(values);
    });

    const worksheet = XLSX.utils.aoa_to_sheet(aoa);
    XLSX.utils.book_append_sheet(workbook, worksheet, name);
  });

  // Write file
  XLSX.writeFile(workbook, outputFile);
  console.log(`Excel file created: ${outputFile}`);
}

module.exports = { generateExcelByLast4Months };

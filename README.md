const XLSX = require("xlsx");

/**
 * Generate an Excel file from query result grouped by report_month
 * using aoa_to_sheet (explicit headers)
 * @param {Array} rows - Result rows from client.query()
 * @param {string} outputFile - Path for output Excel file
 */
function generateExcelByMonth(rows, outputFile = "Report.xlsx") {
  const workbook = XLSX.utils.book_new();

  // Group rows by report_month
  const grouped = rows.reduce((acc, row) => {
    const month = row.report_month; // assumes column is report_month
    if (!acc[month]) acc[month] = [];
    acc[month].push(row);
    return acc;
  }, {});

  // Get column names (from first row)
  const headers = rows.length > 0 ? Object.keys(rows[0]) : [];

  // For each month, add a sheet
  Object.entries(grouped).forEach(([month, data]) => {
    // Build AOA (array of arrays): first row = headers
    const aoa = [
      headers,
      ...data.map(row => headers.map(h => row[h]))
    ];

    const worksheet = XLSX.utils.aoa_to_sheet(aoa);
    XLSX.utils.book_append_sheet(workbook, worksheet, month);
  });

  // Write workbook to file
  XLSX.writeFile(workbook, outputFile);
  console.log(`Excel file created: ${outputFile}`);
}

module.exports = { generateExcelByMonth };

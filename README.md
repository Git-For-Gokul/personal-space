const ExcelJS = require("exceljs");

async function test() {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet("Test");

  ws.addRow(["ID", "Name"]);
  ws.addRow([1, "Alice"]);
  ws.addRow([2, "Bob"]);

  await wb.xlsx.writeFile("test.xlsx");
  console.log("✅ Excel file created: test.xlsx");
}

test();

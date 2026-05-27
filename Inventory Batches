using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;

namespace InventoryManagementSystem
{
    #region Models

    public class InventoryBatch
    {
        // System Fields
        public string RecordId { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public bool IsActive { get; set; }
        public string Checksum { get; set; }

        // Domain Fields (4 Required)
        public string ItemName { get; set; }
        public string BatchNumber { get; set; }
        public int Quantity { get; set; }
        public double UnitPrice { get; set; }

        // Helper to serialize object to a pipe-delimited string
        public string ToFileString()
        {
            return $"{RecordId}|{ItemName}|{BatchNumber}|{Quantity}|{UnitPrice}|" +
                   $"{CreatedAt:O}|{UpdatedAt:O}|{IsActive}|{Checksum}";
        }

        // Helper to deserialize from file string
        public static InventoryBatch FromFileString(string line)
        {
            var parts = line.Split('|');
            if (parts.Length < 9) return null;

            return new InventoryBatch
            {
                RecordId = parts[0],
                ItemName = parts[1],
                BatchNumber = parts[2],
                Quantity = int.Parse(parts[3]),
                UnitPrice = double.Parse(parts[4]),
                CreatedAt = DateTime.Parse(parts[5]),
                UpdatedAt = DateTime.Parse(parts[6]),
                IsActive = bool.Parse(parts[7]),
                Checksum = parts[8]
            };
        }

        // Computes SHA256 hash using only data fields to ensure integrity
        public string CalculateHash()
        {
            string rawData = $"{RecordId}{ItemName}{BatchNumber}{Quantity}{UnitPrice}{CreatedAt:O}{UpdatedAt:O}{IsActive}";
            using (SHA256 sha256Hash = SHA256.Create())
            {
                byte[] bytes = sha256Hash.ComputeHash(Encoding.UTF8.GetBytes(rawData));
                StringBuilder builder = new StringBuilder();
                for (int i = 0; i < bytes.Length; i++)
                {
                    builder.Append(bytes[i].ToString("x2"));
                }
                return builder.ToString();
            }
        }
    }

    #endregion

    #region Services

    public class AuditLogger
    {
        private readonly string _logFilePath;

        public AuditLogger(string logFilePath)
        {
            _logFilePath = logFilePath;
        }

        public void Log(string action, string details)
        {
            try
            {
                string logLine = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] ACTION: {action} | DETAILS: {details}";
                File.AppendAllText(_logFilePath, logLine + Environment.NewLine);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Critical Logging Error: {ex.Message}");
            }
        }
    }

    public class Validator
    {
        public static (bool IsValid, string ErrorMessage) ValidateBatch(string name, string batchNum, string qtyStr, string priceStr)
        {
            if (string.IsNullOrWhiteSpace(name)) return (false, "Item Name cannot be empty.");
            if (string.IsNullOrWhiteSpace(batchNum)) return (false, "Batch Number cannot be empty.");

            if (!int.TryParse(qtyStr, out int qty) || qty < 0)
                return (false, "Quantity must be a valid non-negative integer.");

            if (!double.TryParse(priceStr, out double price) || price <= 0)
                return (false, "Unit Price must be a positive number.");

            return (true, string.Empty);
        }
    }

    public class FileRepository
    {
        private readonly string _dataFilePath;
        private readonly AuditLogger _logger;

        public FileRepository(string dataFilePath, AuditLogger logger)
        {
            _dataFilePath = dataFilePath;
            _logger = logger;
        }

        public List<InventoryBatch> GetAllRaw()
        {
            var records = new List<InventoryBatch>();
            if (!File.Exists(_dataFilePath)) return records;

            try
            {
                var lines = File.ReadAllLines(_dataFilePath);
                foreach (var line in lines)
                {
                    if (string.IsNullOrWhiteSpace(line)) continue;
                    var record = InventoryBatch.FromFileString(line);
                    if (record != null)
                    {
                        // Checksum validation to detect external file tampering
                        if (record.Checksum != record.CalculateHash())
                        {
                            _logger.Log("ERROR", $"Data tampering detected for Record ID: {record.RecordId}!");
                            Console.ForegroundColor = ConsoleColor.Red;
                            Console.WriteLine($"[WARNING] Record {record.RecordId} failed integrity check! Potential corruption.");
                            Console.ResetColor();
                        }
                        records.Add(record);
                    }
                }
            }
            catch (IOException ex)
            {
                _logger.Log("ERROR", $"Failed to read file: {ex.Message}");
                Console.WriteLine($"File read error: {ex.Message}");
            }

            return records;
        }

        public List<InventoryBatch> GetAllActive()
        {
            _logger.Log("READ", "Fetched all active inventory batches.");
            return GetAllRaw().Where(r => r.IsActive).ToList();
        }

        public void SaveAll(List<InventoryBatch> records)
        {
            try
            {
                var lines = records.Select(r => r.ToFileString()).ToArray();
                File.WriteAllLines(_dataFilePath, lines);
            }
            catch (IOException ex)
            {
                _logger.Log("ERROR", $"Failed to write to file: {ex.Message}");
                Console.WriteLine($"File write error: {ex.Message}");
            }
        }

        public void Add(InventoryBatch record)
        {
            var records = GetAllRaw();
            record.Checksum = record.CalculateHash(); // Compute baseline checksum
            records.Add(record);
            SaveAll(records);
            _logger.Log("ADD", $"Added Batch {record.BatchNumber} for {record.ItemName} (ID: {record.RecordId})");
        }

        public bool Update(string id, string name, string batchNum, int qty, double price)
        {
            var records = GetAllRaw();
            var record = records.FirstOrDefault(r => r.RecordId == id && r.IsActive);

            if (record == null)
            {
                _logger.Log("ERROR", $"Update failed. Active Record ID {id} not found.");
                return false;
            }

            record.ItemName = name;
            record.BatchNumber = batchNum;
            record.Quantity = qty;
            record.UnitPrice = price;
            record.UpdatedAt = DateTime.Now;
            record.Checksum = record.CalculateHash(); // Recompute checksum after mutation

            SaveAll(records);
            _logger.Log("UPDATE", $"Updated Record ID: {id}");
            return true;
        }

        public bool SoftDelete(string id)
        {
            var records = GetAllRaw();
            var record = records.FirstOrDefault(r => r.RecordId == id && r.IsActive);

            if (record == null)
            {
                _logger.Log("ERROR", $"Soft delete failed. Record ID {id} not found.");
                return false;
            }

            record.IsActive = false;
            record.UpdatedAt = DateTime.Now;
            record.Checksum = record.CalculateHash(); // Recompute check value for updated state

            SaveAll(records);
            _logger.Log("DELETE_SOFT", $"Soft-deleted Record ID: {id}");
            return true;
        }

        public bool HardDelete(string id)
        {
            var records = GetAllRaw();
            var record = records.FirstOrDefault(r => r.RecordId == id);

            if (record == null)
            {
                _logger.Log("ERROR", $"Hard delete failed. Record ID {id} not found.");
                return false;
            }

            records.Remove(record);
            SaveAll(records);
            _logger.Log("DELETE_HARD", $"Permanently purged Record ID: {id} from disk.");
            return true;
        }
    }

    public class ReportGenerator
    {
        private readonly FileRepository _repo;
        private readonly AuditLogger _logger;

        public ReportGenerator(FileRepository repo, AuditLogger logger)
        {
            _repo = repo;
            _logger = logger;
        }

        // Custom Rule Reporting: Summary analysis and low-stock warning thresholds
        public void GenerateInventorySummaryReport()
        {
            _logger.Log("REPORT", "Generated Financial and Stock Status Report.");
            var activeItems = _repo.GetAllActive();

            Console.WriteLine("\n=======================================================");
            Console.WriteLine("          INVENTORY FINANCIAL & STATUS REPORT          ");
            Console.WriteLine("=======================================================");
            Console.WriteLine($"Total Active Batches: {activeItems.Count}");
            Console.WriteLine($"Total Stock Items:   {activeItems.Sum(i => i.Quantity)} units");
            Console.WriteLine($"Total Asset Value:   ${activeItems.Sum(i => i.Quantity * i.UnitPrice):N2}");
            Console.WriteLine("-------------------------------------------------------");

            // Low Stock Condition Rule (Items with less than 15 units left)
            var lowStockItems = activeItems.Where(i => i.Quantity < 15).ToList();
            if (lowStockItems.Any())
            {
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine("⚠️  LOW STOCK WARNING (Less than 15 units remaining):");
                foreach (var item in lowStockItems)
                {
                    Console.WriteLine($" -> ID: {item.RecordId} | {item.ItemName} (Batch: {item.BatchNumber}) - Only {item.Quantity} left!");
                }
                Console.ResetColor();
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine("✅ All active batches are adequately stocked.");
                Console.ResetColor();
            }
            Console.WriteLine("=======================================================\n");
        }
    }

    #endregion

    #region Main Controller

    class Program
    {
        private static string StorageFolder = "StorageData";
        private static string DataFile = Path.Combine(StorageFolder, "inventory_batches.txt");
        private static string AuditFile = Path.Combine(StorageFolder, "audit_log.txt");

        private static FileRepository _repository;
        private static AuditLogger _logger;
        private static ReportGenerator _reportGenerator;

        static void Main(string[] args)
        {
            InitializeStorage();

            _logger = new AuditLogger(AuditFile);
            _repository = new FileRepository(DataFile, _logger);
            _reportGenerator = new ReportGenerator(_repository, _logger);

            _logger.Log("SYSTEM", "Application launched successfully.");

            bool runProgram = true;
            while (runProgram)
            {
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine("=== INVENTORY BATCH MANAGEMENT SYSTEM ===");
                Console.ResetColor();
                Console.WriteLine("1. Add New Batch");
                Console.WriteLine("2. View All Active Batches");
                Console.WriteLine("3. Search Batches by Item Name");
                Console.WriteLine("4. Update Existing Batch");
                Console.WriteLine("5. Delete Batch (Soft Delete)");
                Console.WriteLine("6. Purge Batch Permanently (Hard Delete)");
                Console.WriteLine("7. Generate Stock & Value Report");
                Console.WriteLine("8. Exit Application");
                Console.Write("Select an operation (1-8): ");

                string selection = Console.ReadLine();
                Console.WriteLine();

                try
                {
                    switch (selection)
                    {
                        case "1": AddBatchWorkflow(); break;
                        case "2": ViewAllBatchesWorkflow(); break;
                        case "3": SearchWorkflow(); break;
                        case "4": UpdateBatchWorkflow(); break;
                        case "5": DeleteWorkflow(hardDelete: false); break;
                        case "6": DeleteWorkflow(hardDelete: true); break;
                        case "7": _reportGenerator.GenerateInventorySummaryReport(); break;
                        case "8":
                            _logger.Log("SYSTEM", "Application closed by user standard routine.");
                            runProgram = false;
                            break;
                        default:
                            Console.ForegroundColor = ConsoleColor.Red;
                            Console.WriteLine("Invalid selection. Please choose an option from 1 to 8.");
                            Console.ResetColor();
                            break;
                    }
                }
                catch (Exception ex)
                {
                    _logger.Log("CRASH_PREVENTED", ex.Message);
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"An unexpected system error occurred: {ex.Message}");
                    Console.ResetColor();
                }
                Console.WriteLine("\nPress any key to clear and return to main menu...");
                Console.ReadKey();
                Console.Clear();
            }
        }

        private static void InitializeStorage()
        {
            if (!Directory.Exists(StorageFolder))
            {
                Directory.CreateDirectory(StorageFolder);
            }
            if (!File.Exists(DataFile))
            {
                File.Create(DataFile).Close();
            }
            if (!File.Exists(AuditFile))
            {
                File.Create(AuditFile).Close();
            }
        }

        private static void AddBatchWorkflow()
        {
            Console.WriteLine("--- Add New Batch Record ---");
            Console.Write("Enter Item Name: "); string name = Console.ReadLine();
            Console.Write("Enter Batch Number: "); string batchNum = Console.ReadLine();
            Console.Write("Enter Initial Quantity: "); string qty = Console.ReadLine();
            Console.Write("Enter Unit Price: "); string price = Console.ReadLine();

            var validation = Validator.ValidateBatch(name, batchNum, qty, price);
            if (!validation.IsValid)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"Validation Failed: {validation.ErrorMessage}");
                Console.ResetColor();
                _logger.Log("VALIDATION_FAIL", $"Failed to add batch. Reason: {validation.ErrorMessage}");
                return;
            }

            var newBatch = new InventoryBatch
            {
                RecordId = Guid.NewGuid().ToString().Substring(0, 8).ToUpper(), // Unique random ID creation
                ItemName = name.Trim(),
                BatchNumber = batchNum.Trim(),
                Quantity = int.Parse(qty),
                UnitPrice = double.Parse(price),
                CreatedAt = DateTime.Now,
                UpdatedAt = DateTime.Now,
                IsActive = true
            };

            _repository.Add(newBatch);
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"Batch record registered successfully! Designated Record ID: {newBatch.RecordId}");
            Console.ResetColor();
        }

        private static void ViewAllBatchesWorkflow()
        {
            var items = _repository.GetAllActive();
            PrintTable(items);
        }

        private static void SearchWorkflow()
        {
            Console.Write("Enter Item Name sequence to search: ");
            string query = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(query))
            {
                Console.WriteLine("Empty query. Search canceled.");
                return;
            }

            var results = _repository.GetAllActive()
                .Where(i => i.ItemName.Contains(query, StringComparison.OrdinalIgnoreCase)).ToList();

            PrintTable(results);
        }

        private static void UpdateBatchWorkflow()
        {
            Console.Write("Enter the Record ID of the batch to update: ");
            string id = Console.ReadLine()?.Trim().ToUpper();

            var items = _repository.GetAllActive();
            var match = items.FirstOrDefault(i => i.RecordId == id);

            if (match == null)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("No active batch with that Record ID discovered.");
                Console.ResetColor();
                return;
            }

            Console.WriteLine($"\nModifying Record found: {match.ItemName} (Batch: {match.BatchNumber})");
            Console.Write($"Enter New Item Name [{match.ItemName}]: "); string name = Console.ReadLine();
            Console.Write($"Enter New Batch Number [{match.BatchNumber}]: "); string batchNum = Console.ReadLine();
            Console.Write($"Enter New Quantity [{match.Quantity}]: "); string qty = Console.ReadLine();
            Console.Write($"Enter New Unit Price [{match.UnitPrice}]: "); string price = Console.ReadLine();

            // Default behavior handles empty entries as keeping baseline values intact
            name = string.IsNullOrWhiteSpace(name) ? match.ItemName : name;
            batchNum = string.IsNullOrWhiteSpace(batchNum) ? match.BatchNumber : batchNum;
            qty = string.IsNullOrWhiteSpace(qty) ? match.Quantity.ToString() : qty;
            price = string.IsNullOrWhiteSpace(price) ? match.UnitPrice.ToString() : price;

            var validation = Validator.ValidateBatch(name, batchNum, qty, price);
            if (!validation.IsValid)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine($"Validation Failed: {validation.ErrorMessage}");
                Console.ResetColor();
                return;
            }

            if (_repository.Update(id, name, batchNum, int.Parse(qty), double.Parse(price)))
            {
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine("Record updated successfully!");
                Console.ResetColor();
            }
        }

        private static void DeleteWorkflow(bool hardDelete)
        {
            string deleteType = hardDelete ? "HARD (PURGE)" : "SOFT (DEACTIVATE)";
            Console.WriteLine($"--- Execute {deleteType} Operations ---");
            Console.Write("Enter the target Record ID: ");
            string id = Console.ReadLine()?.Trim().ToUpper();

            if (string.IsNullOrWhiteSpace(id)) return;

            bool success = hardDelete ? _repository.HardDelete(id) : _repository.SoftDelete(id);

            if (success)
            {
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine($"Operation successful. Record configuration state updated.");
                Console.ResetColor();
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("Action aborted. Target Record ID not found within operation scope.");
                Console.ResetColor();
            }
        }

        private static void PrintTable(List<InventoryBatch> items)
        {
            if (!items.Any())
            {
                Console.ForegroundColor = ConsoleColor.DarkYellow;
                Console.WriteLine("No available data records found.");
                Console.ResetColor();
                return;
            }

            Console.WriteLine(new string('-', 82));
            Console.WriteLine($"| {"ID",-8} | {"Item Name",-20} | {"Batch #",-12} | {"Qty",-6} | {"Price",-10} | {"Last Updated",-12} |");
            Console.WriteLine(new string('-', 82));
            foreach (var item in items)
            {
                Console.WriteLine($"| {item.RecordId,-8} | {Truncate(item.ItemName, 20),-20} | {Truncate(item.BatchNumber, 12),-12} | {item.Quantity,-6} | ${item.UnitPrice,-9:F2} | {item.UpdatedAt:yyyy-MM-dd} |");
            }
            Console.WriteLine(new string('-', 82));
        }

        private static string Truncate(string value, int maxChars)
        {
            return value.Length <= maxChars ? value : value.Substring(0, maxChars - 3) + "...";
        }
    }

    #endregion
}

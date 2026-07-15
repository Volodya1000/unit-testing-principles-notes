```csharp

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using FluentAssertions;
using Xunit;

namespace Book.Chapter6.Listing7_.Functional
{
    public class AuditManager
    {
        private readonly int _maxEntriesPerFile;

        public AuditManager(int maxEntriesPerFile)
        {
            _maxEntriesPerFile = maxEntriesPerFile;
        }

        // Этап 1: Чистая функция для выбора нужного файла из списка имен
        // Работает только с метаданными, не загружая контент
        public FileMeta? GetLastFile(FileMeta[] files)
        {
            if (files.Length == 0)
                return null;

            return SortByIndex(files).Last().file;
        }

        // Этап 2: Чистая функция принятия решения
        // Принимает контент ТОЛЬКО одного файла (или null, если файлов нет)
        public FileUpdate AddRecord(
            FileContent? lastFile,
            string visitorName,
            DateTime timeOfVisit)
        {
            string newRecord = visitorName + ';' + timeOfVisit.ToString("s");

            // Сценарий 1: Файлов еще нет
            if (lastFile == null)
            {
                return new FileUpdate("audit_1.txt", newRecord);
            }

            // Сценарий 2: Файлы есть, проверяем переполнение
            List<string> lines = lastFile.Lines.ToList();

            if (lines.Count < _maxEntriesPerFile)
            {
                lines.Add(newRecord);
                string newContent = string.Join("\r\n", lines);
                return new FileUpdate(lastFile.FileName, newContent);
            }
            else
            {
                int currentFileIndex = GetIndex(lastFile.FileName);
                int newIndex = currentFileIndex + 1;
                string newName = $"audit_{newIndex}.txt";
                return new FileUpdate(newName, newRecord);
            }
        }

        private (int index, FileMeta file)[] SortByIndex(FileMeta[] files)
        {
            return files
                .Select(file => (index: GetIndex(file.FileName), file))
                .OrderBy(x => x.index)
                .ToArray();
        }

        private int GetIndex(string fileName)
        {
            // File name example: audit_1.txt
            string name = Path.GetFileNameWithoutExtension(fileName);
            return int.Parse(name.Split('_')[1]);
        }
    }

    // Легковесная структура: только имя файла
    public class FileMeta
    {
        public readonly string FileName;
        public FileMeta(string fileName) => FileName = fileName;
    }

    public struct FileUpdate
    {
        public readonly string FileName;
        public readonly string NewContent;

        public FileUpdate(string fileName, string newContent)
        {
            FileName = fileName;
            NewContent = newContent;
        }
    }

    // Тяжелая структура: имя + контент
    public class FileContent
    {
        public readonly string FileName;
        public readonly string[] Lines;

        public FileContent(string fileName, string[] lines)
        {
            FileName = fileName;
            Lines = lines;
        }
    }

    public class Persister
    {
        // БЫСТРО: Читаем только имена файлов (мало памяти)
        public FileMeta[] ReadDirectoryFileNames(string directoryName)
        {
            return Directory
                .GetFiles(directoryName)
                .Select(x => new FileMeta(Path.GetFileName(x)))
                .ToArray();
        }

        // МЕДЛЕННО: Загружаем контент конкретного файла
        public FileContent ReadFile(string directoryName, string fileName)
        {
            string filePath = Path.Combine(directoryName, fileName);
            return new FileContent(
                fileName,
                File.ReadAllLines(filePath));
        }

        public void ApplyUpdate(string directoryName, FileUpdate update)
        {
            string filePath = Path.Combine(directoryName, update.FileName);
            File.WriteAllText(filePath, update.NewContent);
        }
    }

    public class ApplicationService
    {
        private readonly string _directoryName;
        private readonly AuditManager _auditManager;
        private readonly Persister _persister;

        public ApplicationService(string directoryName, int maxEntriesPerFile)
        {
            _directoryName = directoryName;
            _auditManager = new AuditManager(maxEntriesPerFile);
            _persister = new Persister();
        }

        public void AddRecord(string visitorName, DateTime timeOfVisit)
        {
            // 1. Получаем список имен (дешевая операция)
            FileMeta[] fileMetas = _persister.ReadDirectoryFileNames(_directoryName);
            
            // 2. Логика решает, какой файл нам нужен
            FileMeta? lastFileMeta = _auditManager.GetLastFile(fileMetas);

            FileContent? lastFileContent = null;

            // 3. Загружаем только ОДИН файл, если он существует
            if (lastFileMeta != null)
            {
                lastFileContent = _persister.ReadFile(_directoryName, lastFileMeta.FileName);
            }

            // 4. Логика формирует обновление
            FileUpdate update = _auditManager.AddRecord(
                lastFileContent, visitorName, timeOfVisit);

            // 5. Сохраняем
            _persister.ApplyUpdate(_directoryName, update);
        }
    }

    public class Tests
    {
        [Fact]
        public void A_new_file_is_created_when_the_current_file_overflows()
        {
            var sut = new AuditManager(3);
            
            // Нам не нужно подавать массив всех файлов.
            // Мы тестируем логику перехода, подавая "последний файл", который переполнен.
            var lastFile = new FileContent("audit_2.txt", new string[]
            {
                "Peter; 2019-04-06T16:30:00",
                "Jane; 2019-04-06T16:40:00",
                "Jack; 2019-04-06T17:00:00"
            });

            FileUpdate update = sut.AddRecord(
                lastFile, "Alice", DateTime.Parse("2019-04-06T18:00:00"));

            Assert.Equal("audit_3.txt", update.FileName);
            Assert.Equal("Alice;2019-04-06T18:00:00", update.NewContent);
            
            // Fluent Assertions style
            update.Should().Be(
                new FileUpdate("audit_3.txt", "Alice;2019-04-06T18:00:00"));
        }
        
        [Fact]
        public void Correctly_finds_the_latest_file()
        {
            var sut = new AuditManager(3);
            var files = new FileMeta[]
            {
                new FileMeta("audit_1.txt"),
                new FileMeta("audit_10.txt"), // Проверка сортировки строк vs чисел
                new FileMeta("audit_2.txt")
            };

            FileMeta? result = sut.GetLastFile(files);

            result.Should().NotBeNull();
            result!.FileName.Should().Be("audit_10.txt");
        }
    }
}
```
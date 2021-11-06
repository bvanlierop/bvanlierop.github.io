
---
title: Mocking The Filesystem Using Abstractions 
date: 2021-03-31 17:00:00 +0100
categories: [testing, design]
tags: [C#, mocks, unit tests, SOLID, dependency injection]
toc: false
---

<img src="{{site.url}}assets/clean.jpg" alt="Cleaning Picture" width="400"/>

# Introduction

On many occasions I come accross code that is tightly coupled to the file system. Here is an example of an import functionality using a CSV file read out:

    {% highlight csharp %}
    using System.Collections;
    using System.IO;

    public class DealershipImporter
    {
        public List<Dealership> Import(string dealershipCatalogCsvPath)
        {
            var dealerships = new List<Dealership>();
            var catalogLines = File.ReadAllLines(dealershipCatalogCsvPath);

            foreach(var csvLine in catalogLines)
            {
                dealerships.Add(new Dealership(csvLine));
            }

            return dealerships;
        }
    }

    [TestFixture]
    public class DealershipImporterTests
    {
        [Test]
        public void ShouldImportDealershipsFromValidCsvFile()
        {
            // Arrange
            const int ExpectedNumberOfDealerships = 5;
            var dealershipImporter = new DealershipImporter();

            // Act
            var dealerships = dealershipImporter.Import(@".\TestFiles\top_five_dealerships.csv");

            // Assert
            Assert.AreEqual(ExpectedNumberOfDealerships, dealerships.Count);           
         }
    }

Class diagram:
<insert here>

On first glance the code looks okay but if you look closer it has some problems:
1. The Importer test is hard to maintain because we have to create and maintain a separate test file
2. The test takes longer because we need to access the disk which takes I/O resources. This adds up if you have thousands of unit tests.
3. The Importer violates the 'D' of the SOLID principle (Dependency Inversion Principle) because we are tightly coupled to System.IO.File class. 

# Improving the testability and design
We can decouple the System.IO dependency by abstracting it away by leveraging a NuGet package called "System.IO.Abstractions". 
It's really useful for these use cases where you want to mock the file system:

    {% highlight csharp %}
    using System.Collections;
    using System.IO.Abstraction;

    public class DealershipImporter
    {
        private IFileSystem fileSystem;

        public DealershipImporter()
        {
            this.fileSystem = new FileSystem(); // uses real System.IO implementation
        }

        public DealershipImporter(IFileSystem fileSystem)
        {
            this.fileSystem = fileSystem;
        }

        public List<Dealership> Import(string dealershipCatalogCsvPath)
        {
            var dealerships = new List<Dealership>();
            var catalogLines = fileSystem.File.ReadAllLines(dealershipCatalogCsvPath);

            foreach(var csvLine in catalogLines)
            {
                dealerships.Add(new Dealership(csvLine));
            }

            return dealerships;
        }
    }

In our test we can then mock out the file system by specifying a test input file up front:

    {% highlight csharp %}
    [Test]
    public void ShouldImportDealershipsFromValidCsvFile()
    {
        // Arrange
        const int ExpectedNumberOfDealerships = 5;
        const string TestFilePath = @"c:\top_file_dealership.csv";

        // Specify test input file up front by using MockFileSystem
        var fileSystem = new MockFileSystem(new Dictionary<string, MockFileData>
        {
            {  TestFilePath, new MockFileData("Dealer 1" + Environment.NewLine + "Dealer 2 ...") }
        });

        // Inject the mock into the Importer class
        var dealershipImporter = new DealershipImporter(fileSystem);

        // Act
        var dealerships = dealershipImporter.Import(TestFilePath);

        // Assert
        Assert.AreEqual(ExpectedNumberOfDealerships, dealerships.Count);
    }

This approach also works great for dealing with System.IO.Directory.Exists(), Create(), Delete() calls and many others.

# Conclusion

As maintainability is very important for code bases, we can raise it by introducing these kind of abstractions.
It helps you to be less dependent on real implementations and in this case, as a side effect, it also speeds up the tests!

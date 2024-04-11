# Internal records processing and report creation AWS lambda function logic 
**Description**
<br>In this example, the logic demonstrated is generic and can handle different data depending on the parameter passed from the banking program. It retrieves the necessary data, converts it into a CSV file, saves a backup, encrypts this file, and transfers it to the bank's SFTP server, essentially for reporting purposes. Additionally, this logic is specifically for an AWS Lambda function.
<br>In *ConvertTransactionToTransactionalData* method i use *using* keyword to prevent any memory leaks and to clean up unmanaged resources.
```csharp
public async Task SendTransactionalDataToBank(DateTime dateToProcess, Solution solution, bool isLambda)
{
    var methodName = $"[{nameof(SendTransactionalDataToBank)}]";
    var currentDateTime = StaticVariables.DateTimeNowEst();
    if (solution != Solution.FirstSolution && solution != Solution.SecondSolution)
    {
        throw new Exception($"{methodName} Solution should be either FirstSolution or SecondSolution");
    }
    
    var solutionName = Enum.GetName(typeof(Solution), solution);
    var programId = solution == Solution.FirstSolution
        ? _internalSettings.BankSettings.FirstSolutionProgramId
        : _internalSettings.BankSettings.SecondSolutionProgramId;

    try
    {
        var transactions = await _remittanceTransactionFundingProvider.GetRemittanceTransactionFundingAsync(dateToProcess, dateToProcess);

        if (!transactions.Any())
        {
            return;
        }

        var merchants = await _merchantProvider.
            GetAllByConditionAsync($" {nameof(Merchant.MerchantID)} IN ({string.Join(",", transactions.Select(t => "'" + t.MerchantId + "'"))})");

        var accounts = await _merchantAccountProvider.
            GetAllByConditionAsync($" {nameof(MerchantAccount.MerchantUid)} IN ({string.Join(",", merchants.Select(m => m.Uid))})");

        var bankCsvProcessor = new BankCsvProcessor(_internalSettings);

        var achDetailId = long.Parse(transactions.First().RefNumber);
        var bankCsvReportBytes = await bankCsvProcessor.ConvertTransactionToTransactionalData(transactions,
            merchants, accounts, currentDateTime, achDetailId, programId);

        var transactionDataLayoutFileName = $"TRN_{_internalSettings.BankSettings.CompanyIdentification}_ACH{programId}_{DateTime.UtcNow:yyyyMMddHHmmss}.csv";
        var encryptedDataLayoutFileName = $"{transactionDataLayoutFileName}.pgp";
        try
        {
            var s3ObjectPath = "Settlement/TransactionalDataLayout/Backup/";

            using var backupStream = new MemoryStream(bankCsvReportBytes);

            _s3Client = new AWS_S3(_internalSettings, _dbContext);
            await _s3Client.UploadObject($"{s3ObjectPath}{transactionDataLayoutFileName.Replace("/", "-")}", backupStream);
        }
        catch (Exception ex)
        {
            await AddLog($"{methodName}[{solutionName}][S3Backup] Exception: {ex.Message}", true);
        }

        var sftpFileLocation = $"/users/{_internalSettings.BankSettings.SFTPUser}/incoming/{encryptedDataLayoutFileName}";

        _bankCommunicationService.CreateSftpClient(isLambda);

        var encryptedFileBytes = _bankCommunicationService.EncryptData(isLambda, bankCsvReportBytes, encryptedDataLayoutFileName, _internalSettings.BankSettings.PGPEncPublic);

        var uploadResult = await _bankCommunicationService.UploadFile(sftpFileLocation, encryptedFileBytes, "Customer Data Layout Upload");


        if (!uploadResult.Success)
        {
            throw new Exception(uploadResult.Message);
        }
    }
    catch (Exception ex)
    {
        await AddLog($"{methodName}[{solutionName}] Exception: {ex.Message}", true);
    }
}
```


```csharp
public async Task<byte[]> ConvertTransactionToTransactionalData(
    IEnumerable<RemittanceTransactionFunding> transactions, 
    IEnumerable<Merchant> merchants, 
    IEnumerable<MerchantAccount> accounts,
    DateTime postDate,
    long achDetailId,
    string programId)
{
    decimal transactionAmountTotal = 0;
    var dateTimeNow = DateTime.UtcNow.ToString("yyyy-MM-dd hh:mm:ss tt");

    var header = new BankTransactionalDataHeader()
    {
        RecordType = HeaderRecordType,
        FileType = _internalSettings.BankSettings.TransactionalDataLayout.FileType,
        Version = _internalSettings.BankSettings.TransactionalDataLayout.Version,
        ProgramType = "ACH",
        CompanyID = _internalSettings.BankSettings.MeCompanyIdentification, // todo: clarify source of this value
        ProgramID = programId,
        FileDateTime = dateTimeNow,
    };

    var customerDataCollection = transactions.Select(item =>
    {
        var merchant = merchants.FirstOrDefault(m => m.MerchantID == item.MerchantId);
        var bankAccount = accounts.FirstOrDefault(b => b.MerchantUid == merchant?.Uid);

        var transactionAmount = item.TransactionAmount;

        transactionAmountTotal += transactionAmount;

        return BankTransactionalDataDetails.FromMerchant(
            merchant,
            DataRecordType,
            _internalSettings.BankSettings.CompanyIdentification,
            programId,
            bankAccount?.AccountNumber,
            item,
            achDetailId
        );
    });

    var trailer = new BankTransactionalDataTrailer
    {
        RecordType = TrailerRecordType,
        RecordCount = customerDataCollection.Count(),
        TransactionAmountTotal = $"{transactionAmountTotal:0.00}"
    };

    using var memoryStream = new MemoryStream();
    using var streamWriter = new StreamWriter(memoryStream);
    using var csvWriter = new CsvWriter(streamWriter, new CsvConfiguration(CultureInfo.InvariantCulture)
    {
        Delimiter = Delimiter,
        HasHeaderRecord = false
    });

    //in order to have new line between records - put to list (or call NextRecord() method)
    csvWriter.WriteRecords(new List<BankTransactionalDataHeader> { header });
    csvWriter.WriteRecords(customerDataCollection);
    csvWriter.WriteRecords(new List<BankTransactionalDataTrailer> { trailer });

    csvWriter.Flush();

    return memoryStream.ToArray();
}
```


# Custom Json Converter
**Description**
<br> *IndividualInfoIntegrationDto* class has complex structure and is needed to be converted from json differently. *BusinessOwnerConverter* inherits from JsonConverter class and implements it's own logic for *IndividualInfoIntegrationDto* objects converting. 

```csharp
    public class BusinessOwnerConverter : JsonConverter
    {
        private const int IndividualCollectionsMaxCount = 4;

        public override void WriteJson(JsonWriter writer, object value, JsonSerializer serializer)
        {
            throw new NotImplementedException();
        }

        public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer)
        {
            var json = JObject.Load(reader);

            foreach (var property in json.Properties().ToList())
            {
                property.Replace(new JProperty(property.Name.ToLower(), property.Value));
            }

            var boDtoType = typeof(BusinessOwnerIntegrationDto);
            var boObjectProperties = boDtoType.GetProperties();

            var individualInfoDto = new IndividualInfoIntegrationDto();
            for (var i = 1; i <= IndividualCollectionsMaxCount; ++i)
            {
                var businessOwner = new BusinessOwnerIntegrationDto();
                foreach (var property in boObjectProperties)
                {
                    var jsonPropertyName = string.Format(property.GetCustomAttribute<JsonPropertyAttribute>().PropertyName, i).ToLower();
                    var objectPropertyType = Nullable.GetUnderlyingType(property.PropertyType) ?? property.PropertyType;

                    var strValue = json[jsonPropertyName]?.ToString()?.Trim();

                    var safeValue = string.IsNullOrEmpty(strValue) ? null : Convert.ChangeType(strValue, objectPropertyType);

                    boDtoType.GetProperty(property.Name)?.SetValue(businessOwner, safeValue);
                }

                individualInfoDto.BusinessOwnerCollection.Add(businessOwner);
            }

            return individualInfoDto;
        }

        public override bool CanConvert(Type objectType)
        {
            return objectType == typeof(IndividualInfoIntegrationDto);
        }
    }
```


# User's permissions validation 
**Description**
<br>All user's data with permissions are stored in the Single Sign-On web applicaiton. This example contains the functionallity that validate user's permission in internall web application calling the SSO application.
*PermissionRequiredFilter* filter contains requests logic to SSO and pass there required permissions taken from the *PermissionAuthorizeAttribute* attribute. 

```csharp
 public class PermissionRequiredFilter : IAsyncAuthorizationFilter
 {
     private const string AuthorizationHeader = "Authorization";
     private const string BearerPrefix = "Bearer ";
     private const string TokenExpired = "Token Expired";

     private readonly IIdentityService _identityService;

     public PermissionRequiredFilter(IIdentityService identityService)
     {
         _identityService = identityService;
     }

     public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
     {
         if (context.ActionDescriptor is ControllerActionDescriptor controllerActionDescriptor)
         {
             if (controllerActionDescriptor.MethodInfo.GetCustomAttribute(typeof(PermissionAuthorizeAttribute)) is PermissionAuthorizeAttribute methodPermissionAttributeData)
             {
                 if (string.IsNullOrEmpty(methodPermissionAttributeData.Permission))
                 {
                     context.Result = new ForbidResult();
                 }
                 else
                 {
                     await PermissionCheckingProcess(context, methodPermissionAttributeData.Permission);
                 }
             }
         }
     }

     private async Task PermissionCheckingProcess(AuthorizationFilterContext context, string permission)
     {
         //Here we expect to receive 200 status code (user has the permission) in order not to block incoming request and process this request
         var permissionCheckStatus = await _identityService.CheckUserPermission(permission);

         switch (permissionCheckStatus)
         {
             case HttpStatusCode.Unauthorized:
                 context.Result = new UnauthorizedObjectResult(TokenExpired);
                 break;
             case HttpStatusCode.NotFound:
                 context.Result = new ForbidResult();
                 break;
         }
     }
 }

[AttributeUsage(AttributeTargets.Method)]
public class PermissionAuthorizeAttribute : AuthorizeAttribute, IFilterFactory
{
    public string Permission { get; set; }

    public bool IsReusable => true;

    public PermissionAuthorizeAttribute(string permission)
    {
        Permission = permission;
    }

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        return (IFilterMetadata) serviceProvider.GetService(typeof(PermissionRequiredFilter));
    }
}
```

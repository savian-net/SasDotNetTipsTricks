# SasDotNetTipsTricks

### Execute SAS Code Using C#
##### Note: The code below is in no particular order and is not directly runable. It is listed here as a guide for how to handle SAS code execution.

```
        public static ConcurrentQueue<SAS.Workspace> SasQueue { get; set; } = new ConcurrentQueue<SAS.Workspace>();



        /// <summary>
        /// Gets a SAS workspace (SAS.exe) from the queue
        /// </summary>
        /// <returns>A SAS workspace object</returns>
        private static Workspace GetSasWorkspace()
        {
            try
            {
                var haveWorkspace = Common.SasQueue.TryDequeue(out Workspace ws);
                if (!haveWorkspace)
                {
                    Log.Info("WARNING: Queue does not have enough workspaces. Adding a new one.");
                    ws = new SAS.Workspace();
                }

                return ws;
            }
            catch (Exception ex)
            {
                Log.Error(DaasMessage.SAS3003.ToString(), ex);
                return null;
            }
        }
        
        

        /// <summary>
        /// Gets a SAS workspace (SAS.exe) from the queue
        /// </summary>
        /// <returns>A SAS workspace object</returns>
        private static Workspace GetSasWorkspace()
        {
            try
            {
                var haveWorkspace = Common.SasQueue.TryDequeue(out Workspace ws);
                if (!haveWorkspace)
                {
                    Log.Info("WARNING: Queue does not have enough workspaces. Adding a new one.");
                    ws = new SAS.Workspace();
                }

                return ws;
            }
            catch (Exception ex)
            {
                Log.Error(DaasMessage.SAS3003.ToString(), ex);
                return null;
            }
        }
        
        
        /// <summary>
        /// Cleans up the SAS workspace object and resets the workspace. 
        /// AKC Suggestion: Make this more expansive and delete the work area vs using SAS tool. I believe it will be faster
        /// </summary>
        /// <param name="ws">SAS Workspace object</param>
        /// <param name="lang">SAS Language Service</param>
        /// <param name="workarea">The workarea to be cleared. This needs to be reinvestigated.</param>
        /// <remarks>Per lead SAS developer in this area (Chris H), the proc datasets is the only way to clear the work library</remarks>
        private async void CleanUpSas(Workspace ws, LanguageService lang, string workarea)
        {
            try
            {
                Log.Info($"Cleaning up : {workarea}");
                Log.Info($"Reset language service and enqueue workspace.");
                lang.Submit("proc datasets library=work nolist kill; run; quit;");
                lang.Reset();
                //20201109 - Decision made to just get a new WS vs trying to reset everything. Issues such as inconsistent quoting, option changes, etc. indicated that it 
                //             was easier to merely kill the old workspace rather than reusing it every time.
                ws = GetSasWorkspace();
                Common.SasQueue.Enqueue(ws);
                Log.Info($"Finished SAS workspace cleanup.");
            }
            catch (Exception ex)
            {
                Log.Error(DaasMessage.SAS3004.ToString(), ex);
            }
        }

        /// <summary>
        /// Starts the SAS workspaces
        /// </summary>
        internal static void StartSasWorkspaces()
        {
            Log.Info($"Starting {Common.AppConfig.SasConfig.Workspaces} SAS workspaces");
            for (int i = 0; i < Common.AppConfig.SasConfig.Workspaces; i++)
            {
                var name = "SASWS_" + i.ToString("D3", Constants.Culture);
                try
                {
                    Log.Info($"Starting workspace: {name}");
                    var ws = new SAS.Workspace
                    {
                        Name = name
                    };
                    var startTime = DateTime.Now;
                    Log.Info($"Workspace {name} started.");
                    var sasWork = Path.Combine(Common.AppConfig.DirectoryConfig.SasWorkDirectory, ws.Name);
                    if (!Directory.Exists(sasWork))
                    {
                        Log.Info($"Creating directory: {sasWork}");
                        Directory.CreateDirectory(sasWork);
                    }

                    SetOptions(ws, name);
                    Log.Info($"Starting workspace: {ws.Name}");
                    Common.SasQueue.Enqueue(ws);
                 }
                catch (Exception ex)
                {
                    Log.Error("Exception starting SAS workspaces.", ex);
                    Log.Error("Note: The following COM libraries must be installed for SAS:");
                    Log.Error("  - Interop.SAS");
                    Log.Error("  - Interop.SASIOMCommon");
                    Log.Error("  - Interop.SASWorkspaceManager");
                }
            }
        }
        

        /// <summary>
        /// Sets SAS options for processing
        /// </summary>
        /// <param name="ws">A SAS workspace to appl options to</param>
        /// <param name="name">The name of the SAS workarea</param>
        private static void SetOptions(SAS.Workspace ws, string name)
        {
            Log.Info("Set the SAS system options");
            var opt = ws.Utilities.OptionService;
            var options = new Dictionary<string, string>();
            var outDir = Path.Combine(Common.AppConfig.DirectoryConfig.SasWorkDirectory, $"{name}");
            if (!Directory.Exists(outDir))
            {
                Log.Info($"SAS work area created: {outDir}");
                Directory.CreateDirectory(outDir);
            }
            options.Add("MEMLIB", "4GB");
            options.Add("NOOVP", "");
            options.Add("MEMSIZE", "MAX");
            options.Add("MEMCACHE", "4");
            options.Add("MEMMAXSZ", "12G");

            var sasOpt = options.Select(p => p.Key).ToArray();
            var sasVals = options.Select(p => p.Value).ToArray();
            Array errIndices = new long[options.Count];
            Array errCodes = new string[options.Count];
            Array errMsgs = new string[options.Count];
            try
            {
                Log.Info($"Setting SAS options for : {outDir}");
                opt.SetOptions(sasOpt, sasVals, out errIndices, out errCodes, out errMsgs);
                foreach (var o in options)
                {
                    Log.Info($"{o.Key,-15}:{o.Value}");
                }
            }
            catch (Exception ex)
            {
                Log.Error("Error when setting SAS options. Most likely, this is caused by doing an " +
                          "Embed Interop Type on the SAS interop references. Check properties and make " +
                          "sure Embed Interop Type is set to false.", ex);
                if (errMsgs != null)
                {
                    foreach (var e in errMsgs)
                    {
                        Log.Error($"SAS error message when setting options: {e.ToString()}");
                    }
                }
                else
                {
                    Log.Error($"SAS error message array is empty.");
                }
            }
        }

       public ActionResult<SasExecutionResult> ExecuteSas(string sasCode, SasMachine sasMachine, string email = null, bool sendStandardEmail = true)
        {
            try
            {
                var machineName = sasMachine.GetAttribute<SasSystemAttribute>().Name;
                sasMachine = SasMachine.SERVERNAME;
                if (sasMachine == SasMachine.Any)
                {
                    Log.Info("SAS Machine not specified. Determining which machine to use.");
                    //sasMachine = Utilities.AssignMachineForProcessing();
                }

                if (machineName != Common.CurrentSystem.SystemName && Common.CurrentSystem.SystemName != "DEV_MACHINE")
                {
                    Log.Info($"Current machine does not match assigned machine: Current[{Common.CurrentSystem.SystemName}], Assigned[{machineName}]");
                    Log.Info($"Submitting remote job {machineName.ToString()}");
                    var result = Utilities.SubmitRemoteSasJobAsync(sasCode, machineName, email);
                    //NOTE: A pssoible exception will be thrown here of:
                    //      A possible object cycle was detected which is not supported. This can either be due 
                    //        to a cycle or if the object depth is larger than the maximum allowed depth of 32.
                    //      This is caused by an issue with the conversion from NewtonSoft.JSON to the .NET serializer.
                    //      It should be fixed by .NET 5
                    return Ok(result);
                }
                Log.Info("Started");
                Log.Info($"Initialize SAS workspace");
                Workspace ws = GetSasWorkspace();
                ws.LanguageService.Reset();
                var reqId = Utilities.GetUniqueId(email);
                Log.Info($"Request Id: {reqId}");
                var subDir = email != null ? email.Split('@')[0] : "_SYSTEM";
                var outfile = Path.Combine(Path.Combine(Common.AppConfig.DirectoryConfig.RequestLogDirectory, subDir), $"{reqId}.zip");
                var exeResult = new SasExecutionResult
                {
                    Start = DateTime.Now,
                    Machine = sasMachine.ToString(),
                    OutfileLocation = outfile
                };
                var lang = ws.LanguageService;
                lang.LineSeparator = "\r\n";
                lang.ResetLogLineNumbers();
                Log.Info($"Submit SAS code: ");
                var code = string.Join(Environment.NewLine, AddHeaderCode(reqId), sasCode, AddFooterCode()); ;
                lang.Submit(code);
                Log.Info("Submitted SAS code.");

                exeResult.SasCode = code;

                /*********************************************************************
                 * IMPORTANT: The following 2 lines MUST be included even though they
                 *            appear to be unused
                 *********************************************************************/
#pragma warning disable CS0219 // Variable is assigned but its value is never used
                const LanguageServiceCarriageControl cc = new LanguageServiceCarriageControl();
                const LanguageServiceLineType lt = new LanguageServiceLineType();
#pragma warning restore CS0219 // Variable is assigned but its value is never used

                lang.FlushLogLines(Common.AppConfig.SasConfig.LogLinesToAnalyze, out Array carriage,
                    out Array lineTypes, out Array lines);
                exeResult.RequestId = reqId;
                exeResult.SasLog = lines.OfType<string>().ToArray();
                exeResult.Errors = exeResult.SasLog.Where(p => p.StartsWith("ERROR:")).ToArray();
                exeResult.Warnings = exeResult.SasLog.Where(p => p.StartsWith("WARNING:")).ToArray();
                //exeResult.SasList = lang.FlushList(Common.AppConfig.SasConfig.ListLinesToAnalyze)
                //                        .Replace("\f", "", true, Constants.Culture);
                exeResult.Finish = DateTime.Now;
                var workarea = Path.Combine(Common.AppConfig.DirectoryConfig.SasWorkDirectory, ws.Name);

                Log.Info($"SAS workarea: {workarea}");
                if (sendStandardEmail)
                {
                    SendZipFile(email, reqId, exeResult);
                }
                CleanUpSas(ws, lang, workarea);
                return exeResult;
            }
            catch (Exception ex)
            {
                Log.Error(Message.SAS3002.ToString(), ex);
                return BadRequest(DaasMessage.SAS3002.ToString() + ex.Message);
            }
        }


        private string AddDaasFooterCode()
        {
            var sb = new StringBuilder();
            sb.AppendLine($"/*" + new string('=', 50) + "*");
            sb.AppendLine($" | CODE ADDED BY SAS WEB SERVICES ");
            sb.AppendLine(" *" + new string('=', 50) + "*/");
            sb.AppendLine($"ods _all_ close;");
            sb.AppendLine($"*{new string('-', 50)} ;");
            return sb.ToString();
        }

        private string AddDaasHeaderCode(string reqId)
        {
            var sb = new StringBuilder();
            sb.AppendLine($"/*" + new string('=', 50) + "*");
            sb.AppendLine($" | CODE ADDED BY SAS WEB SERVICES ");
            sb.AppendLine(" *" + new string('=', 50) + "*/");
            sb.AppendLine($"options fullstimer;");
            sb.AppendLine();
            sb.AppendLine($"%LET SVCSID = {reqId};");
            sb.AppendLine($"%LET SVCSWORK = {Common.AppConfig.DirectoryConfig.RequestLogDirectory};");
            sb.AppendLine();
            var htmlFile = Path.Combine(Common.AppConfig.DirectoryConfig.RequestLogDirectory, reqId + ".html");
            sb.AppendLine($"ods _all_ close;");
            sb.AppendLine($"ods html file='{htmlFile}';");
            sb.AppendLine();
            sb.AppendLine($"*{new string('-', 50)} ;");
            sb.AppendLine();
            return sb.ToString();
        }

```
